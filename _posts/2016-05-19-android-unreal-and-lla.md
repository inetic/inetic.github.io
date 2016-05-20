---
layout: post
title: Low latency audio
subtitle: On Android using Unreal engine
bigimg: #/img/math-pix.jpg
comments: true
---

# Introduction

I've recently had some time and an opportunity to look inside some audio programming. To be honest, I never thought there was much to it, just open a file and read the data while also writing it to some sound output function. Well, yes and no, it turns out that if one is happy with just playing some music, things are that simple. But if one wants to minimize the delay between the event when user presses a button until the sound is played (commonly called lag), there is some trickery that needs to be done, and that is the content of this post.

As the title suggests, we'll be looking at how to minimize the lag on Android while using the [Unreal engine](https://www.unrealengine.com/what-is-unreal-engine-4). Unfortunately, the engine code doesn't support this out of the box, so we'll need to modify the engine's sources as well. The bright side is that the engine does come up with a lot of functionality that we'll use so our changes will be minimal.

# The problems

There are different kinds of APIs one can use for audio programming, but Unreal uses [OpenSL ES](https://en.wikipedia.org/wiki/OpenSL_ES) for Android, which is also the only one supporting LLA. In short, the way this API works is that during an initialization, you pass it a C callback responsible for re-filling sound data to the system. After initialization, whenever you want to play something, you enqueue first few bytes of the sound data and wait for the system to ask you for more data through the C callback function.

The above is the standard approach, and is also the one Unreal uses, but it has one drawback which is the *warm-up* latency. That is, when nothing is being played, the thread inside OpenSL responsible for playing sound goes to a dormant mode to preserve phone's battery. During this mode, it decreases the frequency of polling for new data and sleeps otherwise. To [avoid this warm-up latency](https://developer.android.com/ndk/guides/audio/output-latency.html#warmup-lat), we'll make Unreal engine constantly feed the OpenSL system new bytes, where these bytes shall be all zeros at times when there is silence.

But even with the warm-up problem gone, there are more problems to be addressed. The next one on the list is the size of the buffer we want to pass to OpenSL. The size of it is directly proportional to the period after which the OpenSL thread requests new data. If it's too big, we get high latency again, and if it's too small, we'll hear glithes. For example, in Unreal engine the size of the buffer is fixed to [8192 samples](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/Engine/Public/AudioDecompress.h#L13) (the link only works if you were granted access to Unreal's sources). This size corresponds to ~186ms when played on 44100 sample rate, which is unacceptable for low latency audio.

The last requirement for low latency audio is the proper sample rate, as nicely explained in [this video](https://youtu.be/d3kfEeMZ65c?t=4) if we used a sample rate different to the one which is native for a given device, the sound data would take a "longer path" inside OpenSL before they are played, thus increasing the latency again.

Fortunately, since [Android 4.1](http://mobilepearls.com/labs/native-android-api/ndk/docs/opensles) (see the Performance section), there is Java API to get just these two constants.

# Getting the buffer size and sample rate constants

When an Android application starts, an instance of a class deriving from Android's *NativeActivity* class is instantiated. In Unreal this class is called [*GameActivity*](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Build/Android/Java/src/com/epicgames/ue4/GameActivity.java), and it is where we'll place our Java methods for retrieving those two constants.

```java
  public boolean AndroidThunkJava_HasAudioLowLatencyFeature()
  {
      PackageManager pm = getPackageManager();
      return pm.hasSystemFeature(PackageManager.FEATURE_AUDIO_LOW_LATENCY);
  }
  
  public int AndroidThunkJava_GetAudioOutputFramesPerBuffer()
  {
      AudioManager am = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
    
      String framesPerBuffer
          = am.getProperty(AudioManager.PROPERTY_OUTPUT_FRAMES_PER_BUFFER);
    
      if (framesPerBuffer == null)
      {
          return 0;
      }
    
      try
      {
          return Integer.parseInt(framesPerBuffer);
      }
      catch (NumberFormatException e)
      {
          return 0;
      }
  }
  
  public int AndroidThunkJava_GetAudioNativeSampleRate()
  {
      AudioManager am = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
    
      String sampleRate = am.getProperty(AudioManager.PROPERTY_OUTPUT_SAMPLE_RATE);
    
      if (sampleRate == null)
      {
          return 44100;
      }
    
      try
      {
          return Integer.parseInt(sampleRate);
      }
      catch (NumberFormatException e)
      {
          return 44100;
      }
  }
```

Since the above is a Java API and we'll be using those constants in C++ code, we need to write some **J**ava **N**ative **I**nterface glue. The below code is added into [AndroidJNI.cpp](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/Launch/Private/Android/AndroidJNI.cpp).

```cpp
  bool AndroidThunkCpp_HasAudioLowLatencyFeature()
  {
      bool Result = false;
      if (JNIEnv* Env = FAndroidApplication::GetJavaEnv())
      {
          Result = FJavaWrapper::CallBooleanMethod(
                      Env,
                      FJavaWrapper::GameActivityThis,
                      FJavaWrapper::AndroidThunkJava_HasAudioLowLatencyFeature);
      }
      return Result;
  }
  
  int32 AndroidThunkCpp_GetAudioOutputFramesPerBuffer()
  {
      int32 Result = 0;
      if (JNIEnv* Env = FAndroidApplication::GetJavaEnv())
      {
          Result = FJavaWrapper::CallIntMethod(
                       Env,
                       FJavaWrapper::GameActivityThis,
                       FJavaWrapper::AndroidThunkJava_GetAudioOutputFramesPerBuffer);
      }
      return Result;
  }
  
  int32 AndroidThunkCpp_GetAudioNativeSampleRate()
  {
      int32 Result = 0;
      if (JNIEnv* Env = FAndroidApplication::GetJavaEnv())
      {
          Result = FJavaWrapper::CallIntMethod(
                       Env,
                       FJavaWrapper::GameActivityThis,
                       FJavaWrapper::AndroidThunkJava_GetAudioNativeSampleRate);
      }
      return Result;
  }
```

I've omited some declarations, they can be found in the final diff attached at the end of this post.

# The eternal loop

Now we're mostly finished with changing the Unreal engine sources. There shall be one more change that will save us few more milliseconds, I'll talk about that later because I think it is important to first know what we'll do in the user code.

When attempting to play sound, an Unreal engine user would normally utilize the [USoundWave](https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/Sound/USoundWave/index.html) object. But that is the one that suffers from the *warm-up* problem. Instead, we'll exploit an object called [USoundWaveProcedural](https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/Sound/USoundWaveProcedural/index.html), in particular, we're interested in it's ability to play indefinitely and in its variable [OnSoundWaveProceduralUnderflow](https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/Sound/USoundWaveProcedural/OnSoundWaveProceduralUnderflow/index.html) which is basically another callback called every time the sound thread needs more data.

For testing purposes, I created a small scene inside the Unreal engine editor, then I created an actor object and attached an *OnInputTouchBegin* event to it. Without further ado, here is the corresponding C++ header file:

```cpp
  #pragma once
  
  #include <memory>
  #include <atomic>
  #include <vector>
  #include "GameFramework/Actor.h"
  #include "Sound/SoundWaveProcedural.h"
  #include "ProceduralSoundActor.generated.h"
  
  UCLASS()
  class MYPROJECT_API AProceduralSoundActor : public AActor
  {
      GENERATED_BODY()
    
  public: 
      // Sets default values for this actor's properties
      AProceduralSoundActor();
  
      // Called when the game starts or when spawned
      virtual void BeginPlay() override;
      
      // Called every frame
      virtual void Tick( float DeltaSeconds ) override;
  
      virtual void NotifyActorOnInputTouchBegin(const ETouchIndex::Type FingerIndex);
  
      UPROPERTY(EditAnywhere)
        UShapeComponent* ButtonBox;
  
      UPROPERTY(EditAnywhere)
          UStaticMeshComponent* Mesh;
  
  private:
      void FillAudio(USoundWaveProcedural* Wave, const int32 SamplesNeeded);
      void ReportOutputCharacteristics();
      void SanitizeOutputParams();
  
  private:
      // Constants (Won't change after BeginPlay is called)
      bool   bAndroidHasAudioLowLatencyFeature;
      uint32 OptimalFramesPerBuffer;
      uint32 SampleRate;
      std::unique_ptr<USoundWaveProcedural> SoundStream;
  
      // Modified only inside the sound thread.
      std::vector<uint8> Buffer;
      bool bPlayingSound;
      uint32 PlayCursor;
  
      // Modified in both threads. Clearing the flag indicates
      // to the audio thread that it should start playing.
      std::atomic_flag StartPlaying;
  };
```

And the relevant parts from the implementation.  The following code gets executed shortly after the application starts. Here, we first utilize the JNI functions we created above, then we create and initialize the *USoundWaveProcedural* object for eternal playback.

```cpp
  // Called when the game starts or when spawned
  void AProceduralSoundActor::BeginPlay()
  {
      StartPlaying.test_and_set();
  
      // Making use of our JNI functions.
      bAndroidHasAudioLowLatencyFeature = AndroidThunkCpp_HasAudioLowLatencyFeature();
      OptimalFramesPerBuffer            = AndroidThunkCpp_GetAudioOutputFramesPerBuffer();
      SampleRate                        = AndroidThunkCpp_GetAudioNativeSampleRate();
  
      // This function sets the above constants to default values
      // if they're not set correctly (their values is zero).
      SanitizeOutputParams();
  
      SoundStream.reset(NewObject<USoundWaveProcedural>());
  
      SoundStream->SampleRate = SampleRate;
      SoundStream->NumChannels = 1;
      SoundStream->Duration = INDEFINITELY_LOOPING_DURATION;
      SoundStream->SoundGroup = SOUNDGROUP_Default;
      SoundStream->bLooping = false;
  
      SoundStream->OnSoundWaveProceduralUnderflow.BindLambda
          ([this](USoundWaveProcedural* Wave, const int32 SamplesNeeded) {
              FillAudio(Wave, SamplesNeeded);
          });
  
      UGameplayStatics::PlaySound2D(GetWorld(), SoundStream.get(),
          .5 /* volume multiplier */,
          1  /* pitch multiplier */,
          0);
  
      Super::BeginPlay();
  }
```

This function get's called whenever the user touches our actor, note that the *OnSoundWaveProceduralUnderflow* callback from above get's called from a thread that runs inside OpenSL. Thus - to be safe - we shall interact with it only through this `StartPlaying` variable which is of type `std::atomic_flag`.

```cpp
  void AProceduralSoundActor::NotifyActorOnInputTouchBegin(const ETouchIndex::Type)
  {
      // Clearing the flag indicates to the sound thread that is should start playing.
      StartPlaying.clear();
      UE_LOG(LogTemp, Warning, TEXT("DTS: Start playing"));
  }
```

And finally the juicy part where we generate some audio data. As promised, these data shall be all zeros whenever there is nothing to play, but when the user touches our actor we start feeding it a sine wave for one the duration of one second.

```cpp
  void AProceduralSoundActor::FillAudio(USoundWaveProcedural* Wave,
                                        const int32 SamplesNeeded)
  {
      // Unreal engine uses a fixed sample size.
      static const uint32 SAMPLE_SIZE = sizeof(uint16);
  
      if (!StartPlaying.test_and_set())
      {
          PlayCursor = 0;
          bPlayingSound = true;
      }
  
      // We're using only one channel.
      const uint32 OptimalSampleCount = OptimalFramesPerBuffer;
      const uint32 SampleCount = FMath::Min<uint32>(OptimalSampleCount, SamplesNeeded);
  
      // If we're not playing, fill the buffer with zeros for silence.
      if (!bPlayingSound)
      {
          const uint32 ByteCount = SampleCount * SAMPLE_SIZE;
          Buffer.resize(ByteCount);
          FMemory::Memset(&Buffer[0], 0, ByteCount);
          Wave->QueueAudio(&Buffer[0], ByteCount);
          return;
      }
  
      uint32 Fraction = 1; // Fraction of a second to play.
      float Frequency = 1046.5;
      uint32 TotalSampleCount = SampleRate / Fraction;
  
      Buffer.resize(SampleCount * SAMPLE_SIZE);
      int16* data = (int16*) &Buffer[0];
  
      // Generate a sine wave which slowly fades away.
      for (size_t i = 0; i < SampleCount; ++i)
      {
          float x = (float)(i + PlayCursor) / SampleRate;
          float v = (float)(TotalSampleCount - i - PlayCursor) / TotalSampleCount;
          data[i] = sin(Frequency * x * 2.f * PI) * v * MAX_int16;
      }
  
      PlayCursor += SampleCount;
  
      if (PlayCursor >= TotalSampleCount)
      {
          bPlayingSound = false;
      }
  
      SoundStream->QueueAudio((const uint8*)data, SampleCount * SAMPLE_SIZE);
  }
```

For complete changes done to the Unreal engine, the user code and a sample APK please have a look [here](https://www.dropbox.com/sh/dtzgpztjkgwv6un/AAAjBoTpEorARBace0XJWquCa?dl=0).

# One more change to the engine

The code inside the engine where audio data are being passed to OpenSL is in the file [AndroidAudioSource.cpp](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/Android/AndroidAudio/Private/AndroidAudioSource.cpp). Apart from OpenSL initialization and destruction, there is also the [C callback](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/Android/AndroidAudio/Private/AndroidAudioSource.cpp#L22) which is passed to OpenSL.

In a pseudo code, the callback looks something like this:

```python
  BufferType BackBuffer;
  BufferType Buffer;

  OnRequestBufferCallback()
      Swap(Buffer, BackBuffer)
      Enqueue(Buffer) // This will now start to play.
      PrepareDataInSeparateThread(BackBuffer)
```

The *PrepareDataInSeparateThread* function does one of two things depending on whether the *USoundWave* or the *USoundWaveProcedural* object is being used. In case of *USoundWave* it starts decoding data. This is ideal, because once the *OnRequestBufferCallback* callback is called again, these data shall be ready to be enqueued.

In case of *UsoundWaveProcedural* it simply calls our *OnSoundWaveProceduralUnderflow* callback which we defined above.

Now here is the problem: Say that the user has not yet requested to play audible sound (i.e. silence is being played). That means that last time `OnRequestBufferCallback` was called, the `Enqueue` command enqueued all zeros and the `PrepareDataInSeparateThread` command filled `BackBuffer` with zeros as well. Now imagine that the user requests to play audible audio, the next time the `OnRequestBufferCallback` is called, the `BackBuffer` is first enqueued. But `BackBuffer` most likely contains all zeros and only the next call to `PrepareDataInSeparateThread` shall fill the `BackBuffer` with audible data. Thus we've effectively lost one cycle.

```
      Our callback fills BackBuffer with silence
          ^
          |     User requests audible data
          |        ^
          |        |             Our callback fills BackBuffer with audible data
          |        |                  ^
          |        |                  |
     ---+-+--------+----------------+-+-------------------------+---------> time
        |                           |                           |
        |                           |                           V
        |                           |              Only here are audible data enqueued
        V                           V
    Enqueue silence           Enqueue again (start of new cycle), BackBuffer contains zeros
```

Instead, when `USoundWaveProcedural` is used, what we want is a diagram that looks something like this:

```
                User requests audible data
                   ^
                   |
                   |
                   |
     ---+----------+---------------------+-------------------------------> time
        |                                |
        |                                V
        V                                Enqueue what our callback returns
    Enqueue what our callback returns    (this time audible data)
    (currently silence)
```

The changes required for this behavior to apply can be found in the [UE.diffs](https://www.dropbox.com/sh/dtzgpztjkgwv6un/AAB_h0GYAKXF78MJebjUhLH0a/ue4.11/UE.diffs?dl=0) file (look for changes inside AndroidAudioSource.cpp).

# Related links

[OpenSL ES 1.0.1 specification (pdf)](https://www.khronos.org/registry/sles/specs/OpenSL_ES_Specification_1.0.1.pdf)
[OpenSL ES for Android](http://mobilepearls.com/labs/native-android-api/ndk/docs/opensles/)
[Low-latency audio playback on Android on StackOverflow](http://stackoverflow.com/questions/14842803/low-latency-audio-playback-on-android)
[What latency to expect on different devices](https://source.android.com/devices/audio/latency_measurements.html)

LLA on Android used to require use of at least two OpenSL buffers, [this is no longer the case](https://android.googlesource.com/platform/frameworks/wilhelm/+/92e53bc98cd938e9917fb02d3e5a9be88423791d%5E!/).
