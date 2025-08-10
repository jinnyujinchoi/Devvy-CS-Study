# 제목 없음

# 🎵 BGM & 효과음 전역 관리 흐름 (React + Zustand + Howler.js)

## 0. 개요

- **목표**: 페이지 전환, 새로고침에도 끊기지 않는 BGM + 전역 볼륨 + 효과음 재생
- **BGM** → `HTMLAudioElement`를 직접 사용 (루프, 경로별 변경)
- **효과음(SFX)** → `Howler.js`로 간단 재생
- **전역 상태** → `Zustand(persist)`로 유지, **직렬화 가능한 값만 저장**

---

## 1. 전역 스토어 설계

### 📝 이유

- 페이지 전환 시 인스턴스를 공유해야 함
- 오직 `currentSrc`, `started` 같은 직렬화 가능한 값만 localStorage에 저장
- `audio` 객체 자체는 메모리에서만 유지

### 💡 구현 흐름

1. `audio`(HTMLAudioElement), `currentSrc`, `volumePct`, `started` 상태 보관
2. 볼륨 변경 시 `audio.volume` 즉시 반영
3. `toggleMuteIcon()`으로 음소거 토글

### 📌 코드

```jsx
// src/store/useAudioStore.js
const useAudioStore = create(
  persist(
    (set, get) => ({
      audio: null,
      currentSrc: null,
      started: false,
      volumePct: 50,

      setAudio: (audio) => set({ audio }),
      setCurrentSrc: (src) => set({ currentSrc: src }),
      setStarted: (bool) => set({ started: bool }),

      setVolumePct: (pct) => {
        const v = Math.max(0, Math.min(100, Number(pct) || 0));
        const { audio } = get();
        if (audio) audio.volume = v / 100;
        set({ volumePct: v });
      },

      toggleMuteIcon: () => {
        const { volumePct, setVolumePct, audio } = get();
        if (volumePct === 0) {
          setVolumePct(10);
          if (audio && audio.paused) audio.play().catch(() => {});
        } else {
          setVolumePct(0);
        }
      },
    }),
    {
      name: "audio-storage",
      partialize: (state) => ({
        currentSrc: state.currentSrc,
        started: state.started,
      }),
    }
  )
);
```

---

## 2. BGMProvider로 라우트별 음악 전환

### 📝 이유

- 라우트 이동 시 상황별 BGM 자동 변경
- 같은 곡이면 교체 안 함
- 모바일 정책(autoplay) 대응 → `started`가 true일 때만 재생 시도

### 💡 구현 흐름

1. `location.pathname`으로 다음 BGM src 결정
2. 현재 src와 다르면 새 `Audio` 인스턴스 생성
3. `started && isPlaying`이면 `play()` 시도, 아니면 트랙만 교체
4. 기존 오디오 멈추고 새 오디오로 교체

### 📌 코드

```jsx
// src/components/audio/BGMProvider.jsx
const BGMProvider = ({ isPlaying = true }) => {
  const location = useLocation();
  const {
    audio, currentSrc, setAudio, setCurrentSrc,
    volumePct, started,
  } = useAudioStore();

  useEffect(() => {
    let nextSrc = null;
    const path = location.pathname;

    if (path === "/" || path === "/home") nextSrc = mainTheme;
    else if (path.startsWith("/waiting") || path.startsWith("/game")) nextSrc = gameTheme;
    else if (path.startsWith("/myroom")) nextSrc = roomTheme;

    if (!nextSrc || currentSrc === nextSrc) return;

    const newAudio = new Audio(nextSrc);
    newAudio.loop = true;
    newAudio.volume = (volumePct ?? 50) / 100;

    const commit = () => {
      if (audio) {
        audio.pause();
        audio.currentTime = 0;
      }
      setAudio(newAudio);
      setCurrentSrc(nextSrc);
    };

    if (started && isPlaying) {
      newAudio.play().then(commit).catch((err) => {
        console.warn("BGM 재생 실패:", err);
        commit();
      });
    } else {
      commit();
    }
  }, [location.pathname, currentSrc, audio, setAudio, setCurrentSrc, volumePct, started, isPlaying]);

  useEffect(() => {
    if (audio) audio.volume = (volumePct ?? 50) / 100;
  }, [audio, volumePct]);

  useEffect(() => {
    if (!audio) return;
    if (started && isPlaying) audio.play().catch(() => {});
    else audio.pause();
  }, [audio, started, isPlaying]);

  return null;
};
```

---

## 3. START 버튼으로 autoplay 허용

### 📝 이유

- 모바일/크롬은 **유저 입력 전 자동 재생 불가**
- START 버튼으로 최초 재생 권한 확보

### 💡 구현 흐름

1. `started`가 false면 START 버튼 노출
2. 클릭 시 `setStarted(true)` + BGM 재생
3. 이후에는 버튼 숨김

### 📌 코드

```jsx
// src/pages/LoginPage.jsx
const LogInPage = () => {
  const { started, setStarted } = useAudioStore();

  return (
    <div>
      {!started && (
        <button onClick={() => setStarted(true)}>
          START
        </button>
      )}
      {/* 로그인 UI */}
    </div>
  );
};
```

---

## 4. 효과음(SFX) — Howler 훅

### 📝 이유

- 효과음은 짧고 즉시 재생 → HTMLAudio보다 Howler가 편리
- 개별 사운드 인스턴스를 바로 생성 후 play

### 💡 구현 흐름

1. `soundMap`에 효과음 파일 등록
2. `playSound(key)`로 즉시 재생
3. `Howler.volume`으로 전체 볼륨 변경 가능

### 📌 코드

```jsx
//src/utils/useSound.js
const soundMap = { click, pookie, buy, entry, leave, game_over, countdown, correct, incorrect, grow };

const useSound = () => {
  const playSound = useCallback((key) => {
    const src = soundMap[key];
    if (!src) return;
    new Howl({ src: [src], volume: 0.6, html5: true }).play();
  }, []);

  return { playSound };
};
```

---

## 5. App 연결

### 💡 구현 흐름

1. `BGMProvider`를 최상위(AppContent)에서 렌더
2. 필요 시 `SoundWrapper`로 UI 볼륨 조절
3. 효과음 전역 볼륨 동기화는 `Howler.volume`로 한 번에 적용 가능

### 📌 코드

```jsx
// App.jsx
function AppContent() {
  return (
    <><BGMProvider isPlaying={true} />
      <SoundWrapper showOnRoutes={["/", "/login", "/home", "/waiting*"]} />
      <Router />
    </>
  );
}
```

---

## 6. 오늘 만난 문제 & 해결

| 문제 | 원인 | 해결 |
| --- | --- | --- |
| 새로고침 시 START 버튼 사라짐 | audio 객체 localStorage 직렬화 불가 + 상태 꼬임 | 버튼 조건을 `!started`로만 판단 |
| 라우트 이동 시 BGM 미재생 | autoplay 정책 제한 | `started && isPlaying`일 때만 `play()` 시도 |
| 볼륨 반전 | 슬라이더 UI 방향 반대 | `setVolumePct(100 - value)`로 반전 |

---

## 7. 📦 Howler.js란?

Howler.js는 웹에서 오디오를 간편하게 재생/관리할 수 있는 JS 라이브러리

**주요 특징**

- 다양한 포맷 지원 (mp3, wav, ogg 등)
- 브라우저 간 호환성 처리
- 볼륨, 루프, 페이드, 구간 재생, 전체 음소거 등 API 제공
- HTML5 Audio와 Web Audio API 자동 선택 (환경에 따라 최적 모드 사용)

**예시**

```jsx
//src/utils/useSound/js
import { Howl } from "howler";

const sound = new Howl({
  src: ["/path/to/sound.mp3"],
  volume: 0.8,
  loop: false,
});

sound.play();
```

---

## 8. ⚠️ BGM / 오디오 사용 시 주의사항

### 🎯 1) **브라우저 자동재생 정책**

- 크롬/모바일은 **사용자 입력(클릭 등)** 없으면 `play()` 차단됨
- → **START 버튼** 등으로 최초 재생 권한 확보

### 🎯 2) **전역 인스턴스 관리**

- 페이지 전환 시 오디오 객체를 재생성하면 음악이 끊김
- → 전역 상태(store)에 인스턴스 저장 & 재사용

### 🎯 3) **직렬화 불가능한 객체**

- `Audio`나 `Howl` 객체는 localStorage에 저장 불가
- → persist에서는 직렬화 가능한 값(`src`, `started`, `volume`)만 저장

### 🎯 4) **볼륨 일관성**

- BGM과 효과음 볼륨을 따로 관리하면 불균형 발생
- → `Howler.volume()`으로 전체 효과음 볼륨을 전역과 맞추기

### 🎯 5) **메모리 관리**

- 새로운 BGM 교체 시, 기존 오디오 `pause()` + `currentTime=0`로 해제
- 필요 없는 효과음 인스턴스는 play 후 자동 GC

---

✅ **결론**

- **BGM**은 HTMLAudio로, **SFX**는 Howler로 관리하는 조합이 효율적
- 전역 상태로 인스턴스를 하나만 유지 → 페이지 이동·새로고침에도 끊김 없는 음악
- `started`와 `isPlaying` 조건으로 브라우저 정책에 대응하면 안정적으로 재생 가능

---