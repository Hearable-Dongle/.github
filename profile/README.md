# VocalPoint
A real-time audio processing device that captures sound from multiple sources in noisy environments, applies DSP and AI algorithms for voice filtering and profiling, and streams processed audio directly to hearing aids via Bluetooth.

## Overview
This project implements an audio pipeline that prioritizes voices that matter most to the user. It utilizes either on-device microphones or Wi-Fi-streamed mobile device microphone audio, filtering ambient noise and amplifying important conversations before transmitting to Bluetooth-enabled hearing aids.

## System Architecture
```
┌─────────────────┐     ┌─────────────────┐
│   On-Device     │     │  Phone Mics     │
│   Microphones   │     │  (Wi-Fi Stream) │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │                       │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   Raspberry Pi CM5    │
         │   + Custom PCB        │
         │                       │
         │  ┌─────────────────┐  │
         │  │ Audio Filtering │  │
         │  │ & Enhancement   │  │
         │  └────────┬────────┘  │
         │           │           │
         │           ▼           │
         │  ┌─────────────────┐  │
         │  │ Audio Capture   │  │
         │  │ & Buffering     │  │
         │  └────────┬────────┘  │
         │           │           │
         │           ▼           │
         │  ┌─────────────────┐  │
         │  │ Voice Profiling │  │
         │  │ & Recognition   │  │
         │  └────────┬────────┘  │
         │           │           │
         │           ▼           │
         │  ┌─────────────────┐  │
         │  │ Audio Filtering │  │
         │  │ & Enhancement   │  │
         │  └────────┬────────┘  │
         │           │           │
         └───────────┼───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   Bluetooth Audio     │
         │   Transmission        │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │    Hearing Aids       │
         └───────────────────────┘
```
## Hardware Components

### Raspberry Pi Compute Module 5
- **Purpose**: Main processing unit for audio DSP and ML inference
- **Key Features**: Quad-core ARM Cortex-A76, hardware-accelerated audio processing
- **Memory**: 4GB+ RAM recommended for real-time voice profiling

### Custom PCB
The custom CM5 carrier board contains:
- **MEMS Microphone Array**: 2-4 analog microphones for spatial audio capture
- **Audio Input**: ADC/DAC, converting analog data to four separate I2S busses
- **I2S Multiplexer**: RPi has no analog inputs and contains one I2S channel. The I2S multiplexer will handle the 4 I2S buses.
- **Power Management**: Power regularization for safety
- **GPIO Breakouts**: For expansion and debugging

## Audio Processing Pipeline
### 1. Audio Capture
**On-Device Microphones**
- Capture ambient audio at 48kHz, 16-bit PCM
- Beamforming using a microphone array for directional pickup
- I2S interface to CM5
**Phone Microphones (Wi-Fi Streaming)**
- Phone app streams audio using low-latency Wi-Fi protocol
- Fallback buffering to handle network jitter
### 2. Voice Profiling & Recognition
```
Audio Input Stream
       │
       ▼
┌──────────────────┐
│  Voice Activity  │
│    Detection     │◄──── Distinguishes speech from noise
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Speaker         │
│  Diarization     │◄──── Separates multiple speakers
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Voice Profile   │
│  Matching        │◄──── Compares against stored profiles
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Priority        │
│  Classification  │◄──── Assigns importance scores
└────────┬─────────┘
         │
         ▼
    Filtered Audio
```
**Implementation Approach**:
- **VAD**: Use WebRTC VAD or Silero VAD for speech detection
- **Speaker Embeddings**: Extract voice fingerprints using speaker verification models
- **Profile Database**: Store voice embeddings for important people (family, colleagues)
- **Real-time Matching**: Cosine similarity scoring against stored profiles
- **Adaptive Learning**: Update profiles over time with user feedback
### 3. Audio Filtering & Enhancement
**Multi-Band Processing**
- **Noise Suppression**: RNNoise or Speex for background noise removal
- **Dynamic Range Compression**: Reduce loud sounds, amplify quiet speech
- **Echo Cancellation**: Remove feedback from hearing aids
**Voice Prioritization**
- Apply gain boost to recognized important voices (e.g., +6dB to +12dB)
- Attenuate or gate unrecognized voices based on user settings
- Spatial processing to maintain directional cues
### 4. Bluetooth Audio Transmission
**Protocol**: Bluetooth Classic A2DP or LE Audio (LC3 codec)
- **Latency Target**: <50ms for acceptable lip-sync
- **Codec**: aptX Low Latency, LC3, or AAC
- **Bitrate**: 256kbps for high-quality audio
- **Dual-streaming**: Simultaneous transmission to both hearing aids
## Data Flow Diagram
```
[Mic Array] ──┐
               ├──> [Audio Sync] ──> [VAD] ──> [Speaker ID] ──┐
[Phone/WiFi] ──┘                                              │
                                                              │
                              ┌───────────────────────────────┘
                              │
                              ▼
                     [Priority Router]
                              │
                ┌─────────────┼─────────────┐
                │             │             │
                ▼             ▼             ▼
         [Important]    [Neutral]     [Background]
           +10dB          0dB           -15dB
                │             │             │
                └─────────────┼─────────────┘
                              │
                              ▼
                        [Mix & EQ]
                              │
                              ▼
                      [Compression/Limiting]
                              │
                              ▼
                      [Bluetooth TX] ──> [Hearing Aids]
```
   
## User Workflow
### Initial Setup
1. **Profile Creation**: User trains the system by recording 30-60 seconds of each important person's voice
2. **Hearing Aid Pairing**: Bluetooth pairing with user's hearing aids
### Real-Time Operation
1. System captures audio from all configured sources
2. Voice activity detection identifies speech segments
3. Speaker recognition matches voices against profiles
4. Audio mixer applies priority-based gain structure
5. Processed audio streams to hearing aids with <50ms latency
### Adaptive Learning
- System asks user for feedback on voice recognition accuracy
- Profiles updated automatically with confirmed positive matches
- Background noise models adapt to common environments (home, office, restaurants, outdoors)
## Performance Targets
- **Latency**: <50ms end-to-end (mic to hearing aid)
- **Voice Recognition Accuracy**: >95% for enrolled speakers
- **Battery Life**: 8 hours on portable power
- **Audio Quality**: 48kHz/16-bit with <0.1% THD
- **Bluetooth Range**: 5+ meters line-of-sight
## Future Enhancements
- **Multi-zone Audio**: Prioritize voices based on spatial location
- **Context Awareness**: Automatic profile switching (meeting mode, family mode, etc.)
- **Cloud Sync**: Backup voice profiles across devices
- **Integration**: Connect with calendar for important meeting participants
- **Hearing Aid Direct Streaming**: MFi/ASHA protocol support for direct iOS/Android streaming
