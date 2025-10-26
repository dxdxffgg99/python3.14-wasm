# python3.14-wasm 실행 가이드

브라우저에서 **CPython 3.14**를 WASM으로 로드하고, **DEVTOOL**에서 바로 파이썬 코드를 실행하는 최소 구성입니다.

---

## 폴더 구성

```
/ (서버 루트)
├─ index.html
├─ python.mjs
├─ python.wasm
└─ python314.zip   # 표준 라이브러리(zip 형태)
```

모든 파일은 같은 디렉터리에 둡니다.

---

## 빠른 시작

1. **로컬 서버 실행**

```bash
# Python 내장 서버
python -m http.server 8000

# 또는 Node http-server
npx http-server -p 8000
```

2. 브라우저에서 접속
   `http://localhost:8000` 열기

3. **콘솔에서 실행**

* ``py(`print(123)`)``
* ``py(`import sys; print(sys.version)`)``
  출력은 페이지의 `<pre id="log">` 영역에 표시됩니다.

---

## index.html (핵심 스니펫)

이미 반영되어 있다면 이 절은 건너뛰세요. 아래처럼 전역으로 `Module`과 `py`를 노출해야 콘솔에서 호출할 수 있습니다.

```html
<script type="module">
  import createModule from './python.mjs';
  const log = s => (document.getElementById('log').textContent += s + '\n');

  window.Module = await createModule({
    locateFile: p => p,
    print: log,
    printErr: log,
    noInitialRun: true,
    preRun(mod) {
      mod.FS.createPreloadedFile('/', 'python314.zip', 'python314.zip', true, true);
      mod.ENV.PYTHONHOME = '/';
      mod.ENV.PYTHONPATH = '/python314.zip';
      mod.ENV.PYTHONDONTWRITEBYTECODE = '1';
    },
  });

  window.py = code => window.Module.callMain(['-S','-c', code]);
</script>
```

---

## 사용법

### 콘솔에서 한 줄 실행

```js
py('print("hello")')
py('import math; print(math.sqrt(2))')
```

### 파일 만들어 실행 (가상 파일시스템)

```js
// 스크립트 파일 작성
Module.FS.writeFile('/hello.py', 'print("hi from file")');

// 파일 실행
Module.callMain(['-S', '/hello.py']);
```

### 모듈 임포트

`python314.zip` 안의 표준 라이브러리는 바로 import 가능합니다.

```js
py('import json, statistics; print(statistics.mean([1,2,3]))')
```

---

## 트러블슈팅

* **404: python314.zip**
  파일 경로 확인. `index.html`과 같은 폴더에 두고 서버 루트가 맞는지 점검.

* **SharedArrayBuffer is not defined**
  스레드 빌드/특정 런타임에서 교차-출처 격리 필요. 헤더를 추가하세요.

  ```bash
  npx http-server -p 8000 -c-1 \
    -H "Cross-Origin-Opener-Policy: same-origin" \
    -H "Cross-Origin-Embedder-Policy: require-corp"
  ```

  프록시/실서버(Nginx 등)에서도 동일한 헤더를 설정해야 합니다.

* **Module/py가 콘솔에서 undefined**
  `<script type="module">` 사용 여부, `await createModule(...)` 호출 여부, 로컬 서버 사용 여부 확인.

* **수정한 zip이 반영 안 됨**
  브라우저 캐시 무시(하드 리로드) 또는 쿼리스트링 버전업(`python314.zip?v=2`).

* **네이티브 확장 모듈(.so/.pyd) 임포트 실패**
  WebAssembly CPython은 네이티브 확장 불가. 순수 파이썬 패키지만 사용하세요.

* **네트워크/스레드 제약**
  소켓/스레드 관련 기능은 브라우저 보안 모델과 빌드 옵션에 따라 제한될 수 있습니다. 필요한 경우 위 COOP/COEP 헤더를 적용하고, 기능 가능 여부를 개별적으로 테스트하세요.

---

## 동작 원리 요약

* `python.wasm`은 Emscripten으로 빌드된 CPython 런타임입니다.
* `python.mjs`가 런타임을 로드하고, `preRun`에서 `python314.zip`을 **가상 파일시스템(FS)** 루트(`/`)에 마운트합니다.
* `PYTHONHOME`와 `PYTHONPATH`를 `/`와 `/python314.zip`으로 지정하여 표준 라이브러리 임포트 경로를 확보합니다.
* 실행은 `Module.callMain([...])`로 이루어지며, `-c` 옵션으로 전달된 코드를 인터프리터가 실행합니다.

---

## 확인 체크리스트

* [ ] `http://localhost` 등 **HTTP**로 서비스됨
* [ ] 콘솔에서 `typeof py === "function"`
* [ ] `py('import sys; print(sys.version)')`가 정상 출력됨

---

## 라이선스/크레딧

* CPython 및 표준 라이브러리: 해당 프로젝트 라이선스 따름
* Emscripten/WebAssembly 런타임: 각 라이선스 따름


---
예제 웹사이트 : [이곳을 누르세요](webpython.dxdxffg.com)
