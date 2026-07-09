```markdown
# RAG 파이프라인 E2E 테스트

MINO 위키의 RAG(Retrieval-Augmented Generation) 파이프라인이 임베딩 모델 정합 및 Supabase 설정 변경 후 정상적으로 동작하는지 검증하는 E2E(End-to-End) 테스트 노트입니다.

## 0. 테스트 목적
`gemini-embedding-001` 임베딩 모델 정합과 `SUPABASE_URL` 시크릿 교정 후, GitHub Issue 생성부터 [[팀-위키-자동화-시스템-구성|MINO 위키 자동화 파이프라인]]을 거쳐 Supabase 데이터베이스에 적재되고 최종적으로 `/wiki` 페이지에 노출되기까지 전체 흐름이 원활히 동작하는지 확인합니다.

## 1. 테스트 범위 및 변경 사항
*   **임베딩 모델:** `gemini-embedding-001` 정합
*   **환경 설정:** `SUPABASE_URL` 시크릿 교정

## 2. 테스트용 식별 내용
본 문서는 [[팀-위키-자동화-시스템-구성|MINO 위키 자동화 파이프라인]] 동작 확인을 위해 작성되었습니다.
*   **검색 키워드:** 파이프라인-검증-2026-06-29
*   **참고:** iOS 팀의 [[모듈화-전략-레이어-기능-하이브리드|모듈화 전략]]은 SPM(Swift Package Manager) 로컬 패키지 기반으로 레이어를 분리합니다.
```
