# Live Translation (실시간 동시통역 앱) 설계 명세서

## 1. 개요 (Overview)
본 애플리케이션은 사용자 간의 음성 및 텍스트를 실시간으로 번역하여 전달하는 웹 기반의 동시통역 플랫폼입니다. 별도의 백엔드(Backend) 서버 없이 100% 클라이언트 사이드 기술과 P2P(Peer-to-Peer) 네트워크를 활용하여 구축되었습니다.

## 2. 아키텍처 (Architecture)
> [!NOTE]
> 서버리스(Serverless) P2P 아키텍처를 채택하여 개인정보 유출 위험을 차단하고 유지보수 비용을 최소화했습니다.

- **Frontend**: HTML5, CSS3 (Vanilla), JavaScript (Vanilla)
- **통신 계층 (Network Layer)**: WebRTC / PeerJS (중앙 서버를 거치지 않는 사용자 간 직접 통신)
- **배포 환경 (Hosting)**: Render Static Site (정적 웹 호스팅)

## 3. 핵심 모듈 및 사용 기술 (Core Modules)

### 3.1. 음성 인식 모듈 (STT - Speech-to-Text)
사용자의 음성을 텍스트로 변환하는 모듈로, 두 가지 엔진을 제공합니다.
- **브라우저 내장 엔진 (Web Speech API)**: 설치나 API 키 없이 사용 가능하며 실시간성이 뛰어남.
- **Groq Whisper 엔진**: 압도적인 속도와 높은 정확도 제공. Web Audio API(`AudioContext`, `AnalyserNode`)를 활용한 **침묵 감지(VAD, Voice Activity Detection)** 알고리즘이 적용되어, 사용자가 말을 멈추면 자동으로 녹음을 분할 전송하는 하이브리드 연속 녹음 기능 지원.

### 3.2. 실시간 번역 모듈 (Translation Engine)
인식된 텍스트를 대상 언어로 번역합니다.
- **MyMemory API**: 기본 제공되는 무료 번역 엔진 (집단 지성 기반).
- **Google Cloud Translation**: 고품질 신경망 번역 (사용자 API 키 필요).
- **DeepL API**: 압도적인 자연스러움을 자랑하는 AI 번역 (사용자 API 키 필요).

### 3.3. P2P 통신 모듈 (PeerJS)
- 방장이 방을 생성하면 고유한 **Peer ID**가 발급됩니다.
- 참여자가 해당 ID(혹은 공유된 링크)를 통해 접속하면 WebRTC 기반의 직접 연결(Data Channel)이 수립됩니다.
- 텍스트, 번역 결과, 현재 말하고 있는지의 상태(Speaking Indicator) 등을 실시간으로 송수신합니다.

### 3.4. AI 요약 및 내보내기 모듈 (Export & Summary)
- **AI 요약**: 대화 종료 후 `Groq LLM (llama-3.3-70b-versatile)` API를 호출하여 전체 대화 트랜스크립트를 기반으로 3~4줄의 요약과 주요 Action Item을 한국어로 추출합니다.
- **내보내기**: 추출된 AI 요약과 전체 대화 기록을 PDF 인쇄(브라우저 내장 기능) 또는 Word 문서(msword Blob 생성) 형식으로 내보냅니다.

## 4. 데이터 플로우 (Data Flow)
1. **입력**: 사용자가 마이크에 대고 발화 (Web Speech API 임시 텍스트 표시).
2. **감지 및 전송**: 1.5초간 침묵이 감지되면 MediaRecorder가 해당 음성 청크(Chunk)를 캡처하여 Groq API로 전송.
3. **텍스트화**: Groq API가 텍스트를 반환.
4. **번역**: 반환된 텍스트를 선택된 번역 엔진(MyMemory/Google/DeepL)으로 전송하여 번역본 수신.
5. **공유 및 출력**: 번역된 텍스트를 화면에 표시하고, P2P Data Channel을 통해 상대방 화면으로 전송. 동시에 Web Speech Synthesis를 통해 상대방 언어로 음성 출력(TTS).
