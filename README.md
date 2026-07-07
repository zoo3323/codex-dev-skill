# codex-dev-skill

Claude Code 플러그인. Claude는 **기획자 겸 검수자**, Codex는 **구현자** 역할을 맡는 `codex-dev` 스킬을 제공합니다. Claude가 직접 코드를 짜지 않고, 계획을 세워 Codex에 위임한 뒤 결과를 검수·재요청하는 방식으로 동작합니다.

## 워크플로우

```mermaid
flowchart TD
    A[스킬 시작: /codex-dev] --> B[Codex 권한 자동 허용 확인]
    B --> C[모델 · effort 선택 질문]
    C --> D[태스크 생성 + 계획 수립]
    D --> E{Codex 사용량 여유 있음?}
    E -- 예 --> F["Codex에 구현 위임\n(mcp__codex__codex)"]
    E -- 아니오: 사용량 소진 --> S["Sonnet 5가 직접 구현\n(Agent, model: sonnet)"]
    F --> H[Claude 검수]
    S --> H
    H --> I{문제 있음?}
    I -- 사소한 문제 --> J[Claude가 직접 수정]
    J --> H
    I -- "그 외 문제 (최대 3회)" --> K["Codex에 재요청\n(mcp__codex__codex-reply)"]
    K --> H
    I -- 없음 --> L[완료 보고]
```

### 단계별 설명

1. **권한 자동 허용** — `mcp__codex__codex` 호출마다 승인 팝업이 뜨지 않도록, 처음 실행될 때 `settings.json`의 `permissions.allow`를 스스로 확인하고 필요하면 추가합니다. 별도 설정 없이 설치만 하면 바로 동작합니다.
2. **모델·effort 선택** — 태스크를 만들기도 전에 이번 실행에서 쓸 Codex 모델과 reasoning effort(low/medium/high/xhigh)를 한 번 묻습니다. 이후 모든 Codex 호출에 동일하게 적용됩니다.
3. **계획 수립** — 요구사항을 정리하고, 작업 범위(파일, 완료 기준)를 명확히 한 뒤 진행 상황을 추적할 태스크 4단계(계획/구현/검수/보고)를 만듭니다.
4. **구현 위임** — Codex에 `sandbox: danger-full-access`, `approval-policy: never`로 위임합니다. 작업 규모에 따라 한 번에 위임 / 여러 세션 병렬 위임 / 단계별 순차 위임 중 적절한 방식을 고릅니다.
5. **사용량 소진 시 폴백** — Codex 사용량이 거의 다 찼거나(캐시 기준 99% 이상) 호출이 쿼터 에러로 실패하면, 재시도 없이 즉시 Claude Sonnet 5가 `Agent` 도구로 직접 구현하도록 전환합니다.
6. **검수** — 기본은 가벼운 검수입니다: 테스트/빌드/타입체크 실행 + 요구사항 체크 + diff 훑어보기. 보안 민감 코드나 데이터 손상 위험이 있는 부분만 골라 정독합니다. (검수에 토큰을 다 쓰면 위임으로 아낀 의미가 없기 때문입니다.)
7. **재요청** — 사소한 문제는 Claude가 바로 고치고, 그 외 문제는 같은 Codex 세션에 구체적으로 재요청합니다(부분별 최대 3회).
8. **완료 보고** — 무엇이 바뀌었고 어떻게 확인했는지 간결히 요약합니다.

## 설치

```
/plugin marketplace add zoo3323/codex-dev-skill
/plugin install codex-dev@codex-dev
```

## 사용법

Claude Code에서 다음처럼 실행합니다:

```
/codex-dev 로그인 API에 rate limiting 추가해줘
```

또는 슬래시 명령 없이 실제 구현이 필요한 요청("코드 짜줘", "기능 구현해줘" 등)에도 자동으로 트리거됩니다. 첫 실행 시:

1. Codex 호출 권한이 자동으로 허용됩니다 (한 번만).
2. 이번 실행에 쓸 Codex 모델/effort를 고르는 질문이 뜹니다.
3. 태스크 목록이 생성되고, 계획 → Codex 위임 → 검수 → 보고 순서로 진행 상황이 표시됩니다.

Codex MCP가 연결되어 있지 않으면 스킬을 쓸 수 없다고 안내하며, 사용자가 명시적으로 "네가 직접 짜줘"라고 하면 이 워크플로우를 건너뛰고 Claude가 바로 구현합니다.

## 제거

```
/plugin uninstall codex-dev@codex-dev
```
