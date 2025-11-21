# 프로덕션 에러를 놓치지 않는 법: Sentry 200% 활용 가이드

"Sentry에 에러가 쌓이긴 하는데, 정확히 뭘 봐야 할지 모르겠어요."

프로젝트 초기에 Sentry를 세팅했습니다. 에러가 발생하면 Sentry 대시보드에 잘 쌓이는 것도 확인했습니다. 하지만 솔직히 말하면 그게 전부였습니다. 대시보드에 들어가면 에러들이 쌓여있긴 한데, 정확히 뭘 봐야 할지 몰라서 그냥 방치하고 있었습니다.

그러다 서비스를 직접 사용하면서 몇 가지 문제를 발견했습니다. 특정 버튼을 클릭했을 때 아무 반응이 없거나, 페이지에서 무한 로딩이 발생하거나, 기능이 의도대로 동작하지 않는 경우가 있었습니다. 오랜만에 Sentry에 들어가보니 역시나 에러가 쌓여있었지만, 어떤 것을 먼저 봐야 할지 몰라 구경만 하고 나왔습니다.

며칠간 서비스를 사용하고 팀원들의 사용 모습을 지켜보면서 깨달았습니다. 사소한 에러라도 사용자 신뢰도를 떨어뜨리고 이탈로 이어질 수 있다는 것을요.

이 글에서는 방치되던 Sentry를 제대로 활용하기 위해 진행한 모니터링 개선 과정을 공유합니다. 소스맵 설정부터 컨텍스트 정보 추가, Session Replay 활용, 성능 모니터링까지 실전에서 바로 적용할 수 있는 구체적인 방법을 다룹니다.

참고로 실제 저희 프로젝트 환경이 아닌, 비슷한 미니 프로젝트 환경을 만들어 적용한 내용을 기반으로 기술했습니다.

**이 글의 대상 독자**

- 프로젝트에서 Sentry를 사용 중이거나 도입을 고려하는 개발자
- Sentry를 설치했지만 효과적으로 활용하지 못하고 있는 개발자
- 에러 모니터링을 개선하고 싶은 프론트엔드 개발자

이 글은 React 19, TypeScript, Webpack 환경을 기준으로 작성되었지만, 다른 프레임워크나 번들러를 사용하더라도 전체적인 흐름과 개념은 동일하게 적용할 수 있습니다.

---

## 1. 문제 상황: 에러는 쌓이는데 해결할 수가 없다

Sentry Issues 페이지는 에러 통계, 브라우저/OS 정보, Stack Trace 등 기본 정보를 제공합니다. 하지만 실제 사용하면서 두 가지 문제를 발견했습니다.

### 1.1 난독화된 Stack Trace

프로덕션 환경에서 코드가 번들링되고 압축되면 Stack Trace에 `bundle.js:1:234567` 형태로만 표시됩니다. 어느 파일의 어느 함수에서 에러가 발생했는지 알 수 없어서, 매번 소스맵을 수동으로 확인해야 했습니다.

### 1.2 부족한 컨텍스트

URL이 `/posts`라고 해도 게시글 목록을 불러오다 발생한 건지, 작성 중 발생한 건지, 삭제 중 발생한 건지 알 수 없었습니다. 사용자 행동, 처리 중이던 데이터 같은 컨텍스트 정보가 없으니 원인 파악이 어려웠습니다.

---

## 2. 기본 세팅: Sentry 초기화와 에러 캡처

실험을 위해 React 19, TypeScript, Webpack 환경을 구축했습니다. Axios로 HTTP 통신을 처리하고, JSON Server로 Mock API를 만들었습니다.

### 2.1 Sentry 초기화

애플리케이션 시작 시 Sentry를 초기화합니다.

```typescript
import * as Sentry from "@sentry/react";

export function initSentry() {
  const dsn = process.env.REACT_APP_SENTRY_DSN;
  const environment = process.env.REACT_APP_SENTRY_ENVIRONMENT || "development";

  if (!dsn || dsn === "YOUR_SENTRY_DSN_HERE") {
    console.warn("⚠️ Sentry DSN이 설정되지 않았습니다.");
    return;
  }

  Sentry.init({
    dsn,
    environment,

    integrations: [
      Sentry.browserTracingIntegration(),
      Sentry.replayIntegration({
        maskAllText: false,
        blockAllMedia: false,
      }),
    ],

    tracesSampleRate: environment === "development" ? 1.0 : 0.1,
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,

    beforeSend(event, hint) {
      if (environment === "development") {
        console.log("📤 Sentry에 에러 전송:", event);
      }

      const error = hint.originalException;
      if (error && error.toString().includes("ResizeObserver")) {
        return null;
      }

      return event;
    },

    debug: environment === "development",
  });

  console.log("✅ Sentry 초기화 완료");
}
```

**주요 설정 값 설명**

| 설정                       | 설명                                       | 권장값               |
| -------------------------- | ------------------------------------------ | -------------------- |
| `dsn`                      | Sentry 프로젝트 식별 주소                  | 환경변수로 관리      |
| `environment`              | 환경 구분 (development/staging/production) | 환경별 필터링 가능   |
| `tracesSampleRate`         | 성능 추적 샘플링 비율                      | 개발: 1.0, 운영: 0.1 |
| `replaysSessionSampleRate` | 일반 세션 녹화 비율                        | 0.1 (비용 절감)      |
| `replaysOnErrorSampleRate` | 에러 발생 세션 녹화 비율                   | 1.0 (전체 녹화)      |

### 2.2 Axios 인터셉터 설정

API 요청 중 발생하는 에러를 Sentry로 전송하도록 설정합니다.

```typescript
import axios, { AxiosError } from "axios";
import * as Sentry from "@sentry/react";

export const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL || "http://localhost:4000",
  timeout: 10000,
  headers: {
    "Content-Type": "application/json",
  },
});

api.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => {
    console.error("[API Response Error]", error);

    let errorMessage = "알 수 없는 에러가 발생했습니다.";
    let errorLevel: "error" | "warning" = "error";

    if (error.response) {
      const status = error.response.status;
      switch (status) {
        case 400:
          errorMessage = "잘못된 요청입니다.";
          errorLevel = "warning";
          break;
        case 401:
          errorMessage = "인증이 필요합니다.";
          errorLevel = "warning";
          break;
        case 403:
          errorMessage = "권한이 없습니다.";
          errorLevel = "warning";
          break;
        case 404:
          errorMessage = "요청한 리소스를 찾을 수 없습니다.";
          errorLevel = "warning";
          break;
        case 500:
          errorMessage = "서버 에러가 발생했습니다.";
          break;
        default:
          errorMessage = "알 수 없는 에러가 발생했습니다.";
      }
    } else if (error.request) {
      errorMessage = "네트워크 에러가 발생했습니다.";
    } else {
      errorMessage = `요청 중 에러가 발생했습니다: ${error.message}`;
    }

    Sentry.captureException(error, {
      level: errorLevel,
      tags: {
        errorType: "api_error",
      },
      contexts: {
        api: {
          url: error.config?.url || "unknown",
          method: error.config?.method || "unknown",
          status: error.response?.status || "no response",
        },
      },
      extra: {
        errorMessage,
        requestData: error.config?.data,
        responseData: error.response?.data,
      },
    });

    return Promise.reject(error);
  }
);
```

Axios 에러는 세 가지 타입으로 구분됩니다.

1. `error.response` 존재: 서버가 2xx 외 상태 코드로 응답
2. `error.request`만 존재: 요청은 보냈지만 응답 없음
3. 그 외: 요청 설정 중 에러 발생

400번대 에러는 클라이언트 문제이므로 `warning`으로, 500번대 에러는 서버 문제이므로 `error`로 레벨을 설정했습니다.

### 2.3 Error Boundary 구현

React 컴포넌트 렌더링 에러를 잡기 위해 Error Boundary를 구현합니다.

```typescript
import { Component, ReactNode, ErrorInfo } from "react";
import * as Sentry from "@sentry/react";

interface ErrorBoundaryProps {
  children: ReactNode;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
    };
  }

  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return {
      hasError: true,
      error,
    };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    console.error("ErrorBoundary caught an error:", error, errorInfo);

    Sentry.captureException(error, {
      level: "error",
      tags: {
        errorType: "render_error",
        errorBoundary: true,
      },
      contexts: {
        react: {
          componentStack: errorInfo.componentStack,
        },
      },
    });
  }

  handleReset = (): void => {
    this.setState({
      hasError: false,
      error: null,
    });
  };

  render(): ReactNode {
    const { hasError, error } = this.state;
    const { children } = this.props;

    if (hasError) {
      return (
        <div style={{ padding: "2rem", textAlign: "center" }}>
          <h1 style={{ color: "#dc3545" }}>⚠️ 에러가 발생했습니다</h1>
          <p>죄송합니다. 예상치 못한 오류가 발생했습니다.</p>
          <button onClick={this.handleReset}>다시 시도</button>

          {process.env.NODE_ENV === "development" && error && (
            <details style={{ marginTop: "2rem" }}>
              <summary>에러 상세 정보</summary>
              <pre>{error.toString()}</pre>
            </details>
          )}
        </div>
      );
    }

    return children;
  }
}

export default ErrorBoundary;
```

**Error Boundary 제한사항**

Error Boundary가 잡을 수 없는 에러들이 있습니다.

- 이벤트 핸들러 내부의 에러
- 비동기 코드의 에러
- 서버 사이드 렌더링 에러
- Error Boundary 자체의 에러

이러한 에러는 `try-catch`나 `Promise.catch`로 별도 처리해야 합니다.

---

## 3. 소스맵으로 정확한 에러 위치 찾기

프로덕션 환경에서는 파일 크기 감소, 네트워크 전송량 감소, 코드 난독화를 위해 코드를 번들링하고 압축합니다. 하지만 압축된 코드는 사람이 읽기 어렵습니다.

### 3.1 소스맵이 필요한 이유

원본 코드의 `calculateTotal` 함수는 압축 후 `a` 같은 한 글자로 변환됩니다. 에러 발생 시 Stack Trace에 `bundle.js:1:234567`로 표시되면 원본 코드 위치를 알 수 없습니다.

소스맵은 번들링된 코드를 원본 코드로 매핑해주는 파일입니다. 다음과 같은 장점이 있습니다.

- 에러 발생 위치를 정확하게 파악할 수 있습니다.
- 디버깅 시간이 크게 단축됩니다.
- 원본 코드를 볼 수 있어 에러 원인 파악이 쉬워집니다.

### 3.2 소스맵 생성 설정

Webpack 설정에서 `devtool` 옵션을 사용해 소스맵을 생성합니다.

```javascript
export default (env, argv) => {
  const isDevelopment = argv.mode === "development";

  return {
    devtool: isDevelopment ? "eval-source-map" : "source-map",
  };
};
```

**환경별 소스맵 옵션**

- 개발 환경 (`eval-source-map`): 빠른 리빌드 속도와 정확한 소스맵 제공
- 프로덕션 환경 (`source-map`): 별도 .map 파일 생성, Sentry 업로드용

### 3.3 Sentry Webpack Plugin 설정

소스맵을 Sentry에 업로드하는 가장 간단한 방법은 플러그인을 사용하는 것입니다.

```bash
npm install --save-dev @sentry/webpack-plugin
```

Webpack 설정에 플러그인을 추가합니다.

```javascript
import { sentryWebpackPlugin } from "@sentry/webpack-plugin";

export default (env, argv) => {
  const isDevelopment = argv.mode === "development";

  return {
    entry: "./src/index.tsx",
    output: {
      path: path.resolve(__dirname, "dist"),
      filename: "bundle.js",
      clean: true,
      publicPath: "/",
    },

    devtool: isDevelopment ? "eval-source-map" : "source-map",

    plugins: [
      ...(!isDevelopment
        ? [
            sentryWebpackPlugin({
              org: "your-org-name",
              project: "your-project-name",
              authToken: process.env.SENTRY_AUTH_TOKEN,
            }),
          ]
        : []),
    ],
  };
};
```

### 3.4 Sentry Auth Token 발급

Sentry 대시보드에서 토큰을 발급받습니다.

1. Settings → Account → API → Auth Tokens 이동
2. Create New Token 클릭
3. Scopes 선택: `project:read`, `project:releases`, `org:read`
4. 생성된 토큰을 `.env` 파일에 저장

```env
SENTRY_AUTH_TOKEN=your_token_here
```

**보안 주의사항**: 토큰을 절대 Git에 커밋하지 마세요. `.gitignore`에 `.env` 파일을 추가하세요.

### 3.5 소스맵 적용 효과

프로덕션 빌드 시 플러그인이 자동으로 다음 작업을 수행합니다.

1. 생성된 소스맵 파일을 찾습니다.
2. Sentry에 업로드합니다.
3. 로컬의 소스맵 파일을 삭제합니다.

**적용 전**

```
bundle.js:1:234567
```

**적용 후**

```
handleApiError (src/pages/PostsPage.tsx:42:15)
```

이제 정확한 파일명과 라인 번호를 확인할 수 있습니다.

---

## 4. 컨텍스트 정보로 에러 상황 파악하기

기본 Sentry 세팅만으로는 에러 발생 당시 상황을 충분히 파악하기 어렵습니다. `/posts` 페이지에서 에러가 발생했다고 해도 게시글 목록 조회 중인지, 작성 중인지, 삭제 중인지 알 수 없습니다.

### 4.1 Sentry Context 시스템 구축

현재 컴포넌트와 사용자 행동 정보를 저장하는 시스템을 만들었습니다.

```typescript
interface SentryContextData {
  component?: string;
  feature?: string;
  action?: string;
  additionalData?: Record<string, unknown>;
}

class SentryContext {
  private static context: SentryContextData = {};

  static setContext(data: SentryContextData): void {
    this.context = { ...this.context, ...data };
  }

  static setComponent(component: string): void {
    this.context.component = component;
  }

  static setFeature(feature: string): void {
    this.context.feature = feature;
  }

  static setAction(action: string): void {
    this.context.action = action;
  }

  static setAdditionalData(data: Record<string, unknown>): void {
    this.context.additionalData = { ...this.context.additionalData, ...data };
  }

  static getContext(): SentryContextData {
    return this.context;
  }

  static clear(): void {
    this.context = {};
  }
}

export default SentryContext;
```

### 4.2 커스텀 훅으로 편리하게 사용하기

매번 클래스 메서드를 직접 호출하는 것은 불편하므로 커스텀 훅을 만들었습니다.

```typescript
import { useEffect } from "react";
import SentryContext from "../lib/sentry-context";

interface UseSentryContextOptions {
  component?: string;
  feature?: string;
  action?: string;
  additionalData?: Record<string, unknown>;
}

export function useSentryContext(options: UseSentryContextOptions) {
  useEffect(() => {
    if (options.component) {
      SentryContext.setComponent(options.component);
    }
    if (options.feature) {
      SentryContext.setFeature(options.feature);
    }
    if (options.action) {
      SentryContext.setAction(options.action);
    }
    if (options.additionalData) {
      SentryContext.setAdditionalData(options.additionalData);
    }

    return () => {
      SentryContext.clear();
    };
  }, [
    options.component,
    options.feature,
    options.action,
    options.additionalData,
  ]);

  return {
    setAction: (action: string) => SentryContext.setAction(action),
    setAdditionalData: (data: Record<string, unknown>) =>
      SentryContext.setAdditionalData(data),
  };
}
```

### 4.3 실제 사용 예시

```typescript
import { useSentryContext } from "../hooks/useSentryContext";

function PostsPage() {
  const { setAction, setAdditionalData } = useSentryContext({
    component: "PostsPage",
    feature: "게시글 관리",
  });

  const handleCreatePost = async () => {
    setAction("게시글 작성 버튼 클릭");

    try {
      const newPost = await api.post("/posts", {
        title: "새 게시글",
        content: "내용",
      });

      setAdditionalData({ postId: newPost.data.id });
    } catch (error) {
      console.error("게시글 작성 실패", error);
    }
  };

  const handleDeletePost = async (postId: number) => {
    setAction("게시글 삭제 버튼 클릭");
    setAdditionalData({ postId });

    try {
      await api.delete(`/posts/${postId}`);
    } catch (error) {
      console.error("게시글 삭제 실패", error);
    }
  };

  return (
    <div>
      <button onClick={handleCreatePost}>게시글 작성</button>
      <button onClick={() => handleDeletePost(123)}>게시글 삭제</button>
    </div>
  );
}
```

페이지 진입 시 자동으로 컴포넌트 이름과 기능이 설정됩니다. 버튼 클릭 시 사용자 행동을 기록하고, 에러 발생 시 컨텍스트 정보가 자동으로 Sentry에 전송됩니다.

### 4.4 Axios 인터셉터에 컨텍스트 정보 추가

에러 발생 시 컨텍스트 정보를 함께 전송하도록 인터셉터를 수정합니다.

```typescript
api.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => {
    const sentryContext = SentryContext.getContext();

    let errorMessage = "알 수 없는 에러가 발생했습니다.";
    let errorLevel: "error" | "warning" = "error";

    if (error.response) {
      const status = error.response.status;
      if (status >= 400 && status < 500) {
        errorLevel = "warning";
        errorMessage =
          {
            400: "잘못된 요청입니다.",
            401: "인증이 필요합니다.",
            403: "권한이 없습니다.",
            404: "요청한 리소스를 찾을 수 없습니다.",
          }[status] || "클라이언트 에러가 발생했습니다.";
      } else if (status >= 500) {
        errorMessage = "서버 에러가 발생했습니다.";
      }
    } else if (error.request) {
      errorMessage = "네트워크 에러가 발생했습니다.";
    } else {
      errorMessage = `요청 중 에러가 발생했습니다: ${error.message}`;
    }

    Sentry.captureException(error, {
      level: errorLevel,
      tags: {
        errorType: "api_error",
        component: sentryContext.component,
        feature: sentryContext.feature,
        action: sentryContext.action,
      },
      contexts: {
        api: {
          url: error.config?.url || "unknown",
          method: error.config?.method || "unknown",
          status: error.response?.status || "no response",
        },
        feature: sentryContext,
      },
      extra: {
        errorMessage,
        requestData: error.config?.data,
        responseData: error.response?.data,
        ...sentryContext.additionalData,
      },
    });

    return Promise.reject(error);
  }
);
```

### 4.5 Sentry 대시보드에서 확인하기

이제 에러 정보가 구조화되어 표시됩니다.

**Tags**

- `component`: PostsPage
- `feature`: 게시글 관리
- `action`: 게시글 삭제 버튼 클릭

**Contexts > API**

- `url`: /posts/123
- `method`: DELETE
- `status`: 404

**Extra**

- `errorMessage`: 요청한 리소스를 찾을 수 없습니다
- `postId`: 123

"PostsPage에서 게시글 삭제 중 404 에러가 발생했고, postId는 123번이었구나" 하고 빠르게 상황을 파악할 수 있습니다.

---

## 5. Session Replay로 사용자 행동 재현하기

에러 로그만으로는 파악하기 어려운 것들이 많습니다. 사용자가 정확히 어떤 순서로 행동했는지, 어떤 버튼을 클릭했는지, 화면이 어떻게 변화했는지 말이죠.

Session Replay는 에러 발생 전까지 사용자 행동을 영상처럼 재현해주는 기능입니다.

### 5.1 Session Replay 활성화

Sentry 초기화 시 `replayIntegration`을 추가하면 활성화됩니다.

```typescript
Sentry.replayIntegration({
  maskAllText: false,
  block: [".sensitive-data"],
});
```

민감한 정보를 포함하는 요소에 `sensitive-data` 클래스를 추가합니다.

```tsx
<input
  type='email'
  className='sensitive-data'
  value={email}
  onChange={(e) => setEmail(e.target.value)}
/>
```

사용자가 이메일을 입력해도 Session Replay에는 기록되지 않습니다.

### 5.3 Session Replay 활용하기

Sentry 대시보드에서 에러를 클릭하고 Replays 탭을 선택하면 사용자 행동을 영상처럼 볼 수 있습니다.

**확인 가능한 정보**

- 사용자가 방문한 페이지
- 클릭한 버튼과 입력한 텍스트
- 에러 발생 직전 행동
- 네트워크 요청 타임라인
- 콘솔 로그
- DOM 변경 사항

영상 플레이어처럼 일시정지, 되감기, 빨리감기가 가능합니다. "내 환경에서는 재현이 안 되는데요" 문제가 사라집니다. 영상으로 보면 답이 보이기 때문입니다.

## 6. 실전 운영 노하우

Sentry를 효과적으로 활용하기 위한 실전 팁을 공유합니다.

### 6.1 에러 레벨 구분 전략

Sentry는 5가지 레벨을 제공합니다. 명확한 기준을 세워야 효과적으로 분류할 수 있습니다.

| 레벨      | 사용 시기                      | 예시                               |
| --------- | ------------------------------ | ---------------------------------- |
| `fatal`   | 서비스 전체가 멈추는 상황      | 앱 크래시, 메모리 부족             |
| `error`   | 기능이 동작하지 않는 경우      | API 500 에러, 데이터 저장 실패     |
| `warning` | 문제는 있지만 우회 가능한 경우 | API 400 에러, deprecated 함수 사용 |
| `info`    | 중요한 이벤트 기록             | 결제 완료, 회원가입                |
| `debug`   | 개발 중 디버깅용               | 프로덕션에서는 사용 안 함          |

### 6.2 불필요한 에러 필터링하기

초반에는 에러가 너무 많아서 중요한 것을 놓칠 수 있습니다. 해결할 수 없는 에러는 필터링합니다.

```typescript
beforeSend(event, hint) {
  const error = hint.originalException;

  // ResizeObserver 에러 필터링
  if (error?.toString().includes("ResizeObserver")) {
    return null;
  }

  // 브라우저 확장 프로그램 에러 필터링
  if (event.exception?.values?.[0]?.stacktrace?.frames?.some(
    frame => frame.filename?.includes("chrome-extension://")
  )) {
    return null;
  }

  // 요청 취소 에러 필터링
  if (error?.message?.includes("canceled")) {
    return null;
  }

  return event;
}
```

### 6.3 Alert 설정 전략

에러 발생마다 알림을 받으면 피곤합니다. 중요도에 따라 차등 적용합니다.

**Slack 즉시 알림**

- `level`이 `fatal`인 에러
- 10분간 같은 에러 50회 이상 발생 (급증)
- 결제, 로그인 같은 핵심 기능 에러

**이메일 일일 요약**

- 처음 발생한 에러
- 재발한 에러
- 에러 트렌드 리포트

### 6.4 에러 우선순위 판단하기

모든 에러를 동시에 해결할 수는 없습니다. 빈도와 심각도로 우선순위를 정합니다.

| 심각도 \ 빈도 | 높음           | 낮음                 |
| ------------- | -------------- | -------------------- |
| **높음**      | 🔴 최우선 해결 | 🟡 빠른 시일 내 해결 |
| **낮음**      | 🟡 개선 필요   | 🟢 모니터링          |

### 6.5 Release 추적하기

어떤 버전에서 에러가 발생했는지 추적하면 빠르게 대응할 수 있습니다.

```typescript
Sentry.init({
  dsn,
  environment,
  release: `my-app@${process.env.REACT_APP_VERSION}`,
});
```

**활용 방법**

- "v1.2.3 배포 후 에러가 급증했네?" 패턴 발견
- 문제 있는 버전 빠르게 롤백
- 에러 수정 후 실제로 해결됐는지 확인

### 6.6 비용 최적화하기

Sentry는 이벤트 수에 따라 비용이 발생합니다. 샘플링 비율을 조정해서 비용을 절감합니다.

```typescript
Sentry.init({
  // 개발: 100% 추적, 운영: 10% 추적
  tracesSampleRate: environment === "development" ? 1.0 : 0.1,

  // 일반 세션: 10%, 에러 세션: 100%
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

**비용 절감 팁**

- 같은 에러 100번 발생해도 1개 이슈로 카운트됨
- 개발 환경에서는 Sentry에 전송하지 않기
- 샘플링 비율을 프로덕션 환경에 맞게 조정하기

### 6.7 Error Boundary 사용 시 주의사항

Error Boundary는 렌더링 에러만 잡습니다. 다음은 잡을 수 없습니다.

- 이벤트 핸들러 내부의 에러
- 비동기 코드의 에러
- 서버 사이드 렌더링 에러

이러한 경우는 `try-catch`나 `Promise.catch`로 별도 처리합니다.

```typescript
function MyComponent() {
  const handleClick = () => {
    try {
      dangerousOperation();
    } catch (error) {
      Sentry.captureException(error);
      alert("에러가 발생했습니다");
    }
  };

  return <button onClick={handleClick}>클릭</button>;
}
```

---

## 마무리

Sentry를 처음 도입했을 때의 막막함에서 시작해서, 단계별로 개선하면서 배운 것들을 정리했습니다.

**핵심 포인트 요약**

- 소스맵을 업로드하면 에러 추적이 훨씬 쉬워집니다.
- 컨텍스트 정보를 추가하면 에러 상황을 빠르게 파악할 수 있습니다.
- Session Replay를 사용하면 에러 재현이 쉬워집니다.
- 에러 레벨, 필터링, Alert 설정으로 효율적으로 운영할 수 있습니다.

무엇보다 중요한 것은 도구를 설치만 하는 게 아니라 실제로 사용하는 것입니다. 매일 에러를 확인하고, 팀과 공유하고, 함께 개선해나가는 문화를 만드는 것이 가장 큰 차이를 만듭니다.

작은 것부터 시작해서 하나씩 개선해나가면, 어느 순간 에러 모니터링이 개발 과정에서 빼놓을 수 없는 부분이 되어 있을 것입니다.

---

## 참고 자료

- [Sentry 공식 문서](https://docs.sentry.io/)
- [Sentry React SDK](https://docs.sentry.io/platforms/javascript/guides/react/)
- [소스맵 업로드 가이드](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Session Replay 문서](https://docs.sentry.io/product/session-replay/)
