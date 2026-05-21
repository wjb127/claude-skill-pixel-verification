# verify-pixels — 광고 픽셀 / GA4 / GTM 설치 검증 + 증거 확보

광고/분석 스크립트가 **실제로 동작하는지** Playwright로 검증하고, 클라이언트 반박 못할 수준의 **스크린샷 + 네트워크 증거 패키지**를 생성한다.

호출 예시:
- `/verify-pixels` — 프로젝트 자동 분석 + 전체 검증
- `/verify-pixels --url https://example.com` — 외부 URL 검증
- `/verify-pixels meta only` — Meta Pixel만
- `/verify-pixels ga4 only` — GA4만
- `/verify-pixels --quick` — 핵심 페이지 1개만 (PageView 검증)

---

## 2026년 컨텍스트 (반드시 인지)

**AI 생성 트래킹 코드의 흔한 오류 (50%+ 실패)**:
1. 잘못된/플레이스홀더 Pixel ID, Measurement ID (`GA_MEASUREMENT_ID`, `XXXXXXXX` 그대로 남음)
2. UA(Universal Analytics) 패턴을 GA4에 적용 (event-based 모델 무시)
3. Config 태그보다 Event 태그가 먼저 fire (session_id, traffic source 누락)
4. **Consent Mode V2 신호 누락** — 2026년 EEA/UK 강제. ad_storage/analytics_storage/ad_user_data/ad_personalization 4개 신호
5. PageView 중복 발사 (gtag + GTM 둘 다 설치, fbq 두 번)
6. 일부 페이지만 설치 (홈만 OK, /pricing 누락 패턴)
7. Custom parameter를 Custom Dimension으로 등록 안 함 → 리포트에 안 보임
8. Measurement Protocol에 debug_mode / engagement_time_msec 누락
9. CAPI(Conversions API) 미연동 — 클라이언트만 → iOS/AdBlock에서 20~35% 누락
10. CAPI/Pixel `event_id` 불일치 → 중복 카운트 → 데이터 인플레이션

**Pixel Helper 그린 ≠ "트래킹 완료"**:
- Pixel Helper는 "브라우저에서 요청 떠남"까지만 보장
- "Meta 서버 수신 + 처리 + 어트리뷰션 사용"은 Events Manager Test Events에서만 확인
- 따라서 검증은 **3계층** 필요: 네트워크 캡처 + Pixel Helper/Tag Assistant + Test Events/DebugView

**에이전시 감사 통계**: 초기 셋업 **40~60%가 파라미터 문제**. CPA 2~3배 악화.

---

## 검증 대상 (자동 감지)

다음 도메인/패턴을 자동 매칭:

| 플랫폼 | 요청 패턴 | 확인 항목 |
|---|---|---|
| GA4 | `google-analytics.com/g/collect`, `analytics.google.com/g/collect`, `region1.google-analytics.com` | tid (G-XXXXX), en (event name), cid, sid, dl (page location), tid format `G-` |
| Google Ads | `googleadservices.com/pagead/conversion`, `googleads.g.doubleclick.net` | conversion label, value, currency |
| GTM | `googletagmanager.com/gtm.js?id=GTM-` | GTM-XXXXX ID format |
| Meta Pixel | `facebook.com/tr`, `facebook.com/tr/`, `connect.facebook.net/en_US/fbevents.js` | id (15~16자리), ev (event name), cd[content_ids], cd[value], eid (event_id) |
| TikTok | `analytics.tiktok.com/api/v2/pixel`, `analytics.tiktok.com/i18n/pixel/events.js` | sdkid, event, properties |
| Naver | `wcs.naver.net/wcslog.js`, `wcs.naver.com/wcslog.gif` | _nasa, _naa |
| Kakao | `t1.daumcdn.net/kas/static/kp.js`, `analytics.ad.daum.net` | kakaoPixelId |
| Microsoft | `bat.bing.com/bat.js`, `bat.bing.com/action/0` | ti (Pixel ID) |
| Hotjar | `static.hotjar.com/c/hotjar-` | siteId |
| Clarity | `clarity.ms/tag/` | projectId |

---

## 실행 흐름

### Step 1 — 프로젝트 / URL 분석

**경우 A: 프로젝트 컨텍스트 있음** (`pwd`로 감지)
- Next.js: `next.config.js` + `app/layout.tsx` / `pages/_app.tsx` / `pages/_document.tsx` 검사
- Astro: `astro.config.mjs` + 레이아웃 컴포넌트 검사
- 정적 HTML: `<head>` 검사
- 배포 URL 감지: `vercel.json`, `.vercel/project.json`, README

**경우 B: URL만 주어짐** (`--url` 옵션)
- 직접 fetch해서 HTML 분석

### Step 2 — 정적 코드 검사 (Source Audit)

다음을 grep / 정규식으로 확인:

```bash
# GA4 Measurement ID 패턴
grep -rE "G-[A-Z0-9]{8,12}" src/ public/ 2>/dev/null

# GTM Container ID
grep -rE "GTM-[A-Z0-9]{6,8}" src/ public/ 2>/dev/null

# Meta Pixel ID (15~16자리 숫자)
grep -rE "fbq\\(['\"]init['\"], ['\"][0-9]{14,17}['\"]" src/ public/ 2>/dev/null

# 플레이스홀더 잔존 확인 (절대 안 됨)
grep -rE "GA_MEASUREMENT_ID|YOUR_PIXEL_ID|XXXXXXXX|G-XXXXXXX|GTM-XXXXXX" src/ public/

# Consent Mode V2 신호
grep -rE "consent\\s*\\(\\s*['\"]default['\"]|ad_storage|analytics_storage" src/ public/
```

**경고 발동 조건**:
- 플레이스홀더 그대로 남아있으면 즉시 RED 플래그
- GA4 ID가 `UA-`로 시작 (Universal Analytics 패턴) → RED
- gtag + GTM 동시 발견 → 중복 위험 YELLOW
- Consent Mode V2 신호 누락 (EEA 타겟이면 RED)

### Step 3 — Playwright 동적 검증

#### 3-1. 네트워크 요청 캡처 스크립트

```ts
import { test, expect, Page } from '@playwright/test';
import fs from 'node:fs';
import path from 'node:path';

const TRACKERS = {
  ga4: /google-analytics\.com\/g\/collect|analytics\.google\.com\/g\/collect|region\d+\.google-analytics\.com/,
  gtm: /googletagmanager\.com\/gtm\.js/,
  googleAds: /googleadservices\.com\/pagead\/conversion|googleads\.g\.doubleclick\.net/,
  metaPixel: /facebook\.com\/tr|connect\.facebook\.net\/en_US\/fbevents\.js/,
  tiktok: /analytics\.tiktok\.com/,
  naver: /wcs\.naver\.(net|com)/,
  kakao: /t1\.daumcdn\.net\/kas|analytics\.ad\.daum\.net/,
  bing: /bat\.bing\.com/,
};

interface TrackerHit {
  platform: string;
  url: string;
  method: string;
  status: number;
  timestamp: number;
  params: Record<string, string>;
  postData?: string;
}

async function captureTrackers(page: Page, route: string, eventDir: string) {
  const hits: TrackerHit[] = [];

  page.on('request', async (req) => {
    const url = req.url();
    for (const [platform, pattern] of Object.entries(TRACKERS)) {
      if (pattern.test(url)) {
        const u = new URL(url);
        const params: Record<string, string> = {};
        u.searchParams.forEach((v, k) => { params[k] = v; });
        hits.push({
          platform,
          url,
          method: req.method(),
          status: 0,
          timestamp: Date.now(),
          params,
          postData: req.postData() || undefined,
        });
      }
    }
  });

  page.on('response', (res) => {
    const url = res.url();
    const hit = hits.find(h => h.url === url && h.status === 0);
    if (hit) hit.status = res.status();
  });

  await page.goto(route, { waitUntil: 'networkidle' });
  await page.waitForTimeout(2000); // 지연 fire 픽셀 대비 (예외적 허용)

  // 스크린샷 + 네트워크 패널 저장
  await page.screenshot({ path: path.join(eventDir, `${route.replace(/\//g, '_')}.png`), fullPage: true });
  fs.writeFileSync(path.join(eventDir, `${route.replace(/\//g, '_')}.json`), JSON.stringify(hits, null, 2));

  return hits;
}
```

#### 3-2. 검증 assertion (Happy/Sad/Bad/Ugly)

```ts
test.describe('GA4 검증', () => {
  test('Happy: 홈에서 page_view 1회만 fire', async ({ page }) => {
    const hits = await captureTrackers(page, '/', './evidence/ga4-home');
    const pageViews = hits.filter(h => h.platform === 'ga4' && h.params.en === 'page_view');
    expect(pageViews.length, '중복 PageView 발견').toBe(1);
    expect(pageViews[0].status, 'GA4 응답 코드 200/204 아님').toBeLessThan(300);
    expect(pageViews[0].params.tid, 'GA4 Measurement ID 누락').toMatch(/^G-[A-Z0-9]+$/);
  });

  test('Sad: 모든 핵심 페이지에 GA4 설치됨', async ({ page }) => {
    const routes = ['/', '/about', '/pricing', '/contact'];
    for (const r of routes) {
      const hits = await captureTrackers(page, r, `./evidence/ga4-${r.replace(/\//g, '_')}`);
      const found = hits.find(h => h.platform === 'ga4');
      expect(found, `${r}에 GA4 미설치`).toBeDefined();
    }
  });

  test('Bad: Measurement ID에 플레이스홀더 없음', async ({ page }) => {
    const hits = await captureTrackers(page, '/', './evidence/ga4-id-check');
    const ga4 = hits.find(h => h.platform === 'ga4');
    expect(ga4?.params.tid).not.toMatch(/XXXX|MEASUREMENT_ID|YOUR_ID/i);
  });
});

test.describe('Meta Pixel 검증', () => {
  test('Happy: PageView fire + 200 응답', async ({ page }) => {
    const hits = await captureTrackers(page, '/', './evidence/meta-home');
    const pv = hits.filter(h => h.platform === 'metaPixel' && /PageView/i.test(h.params.ev || ''));
    expect(pv.length, 'Meta PageView 없음').toBeGreaterThanOrEqual(1);
    expect(pv.length, 'Meta PageView 중복').toBeLessThanOrEqual(1);
    expect(pv[0].params.id, 'Pixel ID 누락').toMatch(/^\d{14,17}$/);
    expect(pv[0].status).toBeLessThan(300);
  });

  test('Bad: Lead event는 폼 제출에서만 fire (페이지 로드에선 X)', async ({ page }) => {
    const hits = await captureTrackers(page, '/', './evidence/meta-lead-not-on-load');
    const leadOnLoad = hits.find(h => h.platform === 'metaPixel' && /Lead/i.test(h.params.ev || ''));
    expect(leadOnLoad, 'Lead가 페이지 로드에서 잘못 fire됨').toBeUndefined();
  });
});

test.describe('Consent Mode V2 (EEA 대상)', () => {
  test('초기 consent default 신호 존재', async ({ page }) => {
    let consentDefault = false;
    page.on('console', msg => {
      if (msg.text().includes('consent') && msg.text().includes('default')) consentDefault = true;
    });
    await page.addInitScript(() => {
      const orig = (window as any).gtag;
      (window as any).gtag = function(...args: any[]) {
        if (args[0] === 'consent' && args[1] === 'default') console.log('[consent default]', JSON.stringify(args[2]));
        return orig?.apply(this, args);
      };
    });
    await page.goto('/', { waitUntil: 'networkidle' });
    // Consent Mode V2는 페이지에 consent_state HTML element 또는 dataLayer 검사
    const dataLayer = await page.evaluate(() => (window as any).dataLayer || []);
    const hasConsent = dataLayer.some((e: any) => e[0] === 'consent' || e.event === 'consent');
    expect(hasConsent || consentDefault, 'Consent Mode V2 신호 없음').toBe(true);
  });
});
```

#### 3-3. 시나리오별 이벤트 검증

핵심 사용자 흐름을 자동 실행 + 그 시점 네트워크 캡처:

| 시나리오 | 트리거 | 기대 이벤트 |
|---|---|---|
| 홈 진입 | `page.goto('/')` | GA4 page_view, Meta PageView (각 1회) |
| 폼 제출 | `getByTestId('submit').click()` | GA4 generate_lead, Meta Lead |
| 결제 완료 | `/order/success` 진입 | GA4 purchase (transaction_id 포함), Meta Purchase (value/currency 포함) |
| 전화 클릭 | `tel:` 링크 클릭 | GA4 phone_click (커스텀) |
| 카카오 클릭 | 채널 링크 클릭 | GA4 kakao_click (커스텀) |

각 시나리오에서:
- 네트워크 요청 캡처 → JSON 저장
- 풀 페이지 스크린샷 → PNG
- 콘솔 로그 캡처 → 에러 0개여야 함

### Step 4 — 외부 검증 도구 스크린샷 (선택, 수동 안내)

Pixel Helper / Tag Assistant는 Chrome 확장이라 Playwright headless에서 안 됨. 다음 중 하나:

**옵션 A — Playwright headed + 확장 로드** (실험적):
```ts
const ctx = await chromium.launchPersistentContext('/tmp/profile', {
  headless: false,
  args: [
    '--disable-extensions-except=/path/to/pixel-helper',
    '--load-extension=/path/to/pixel-helper',
  ],
});
```

**옵션 B — 사용자에게 명확한 가이드 (권장)**:
사용자에게 다음 안내 + 스크린샷 받기:
```
Pixel Helper / Tag Assistant 검증은 자동화 어려움 (Chrome 확장 + Facebook 로그인 필요).
다음 단계로 직접 확인 후 스크린샷 저장해줘:
1. Chrome에서 https://<url> 접속
2. Meta Pixel Helper 클릭 → 사이드 패널 캡처
3. ./evidence/manual/pixel-helper.png 로 저장
4. GA4 DebugView (https://analytics.google.com → 관리자 → DebugView) 열고
   해당 사이트에서 클릭 몇 번 한 뒤 실시간 이벤트 캡처
5. ./evidence/manual/ga4-debugview.png 로 저장
```

### Step 5 — 증거 패키지 생성 (HTML 리포트)

`tracking-audit-report-YYYYMMDD.html` 생성:

```
프로젝트 루트/
└── tracking-audit-report-20260522.html
└── evidence/
    ├── ga4-home.png             (풀페이지 스크린샷)
    ├── ga4-home.json            (캡처된 네트워크 요청)
    ├── ga4-pricing.png
    ├── meta-home.png
    ├── meta-home.json
    ├── consent-check.png
    └── manual/
        ├── pixel-helper.png     (사용자 제공)
        └── ga4-debugview.png    (사용자 제공)
```

HTML 리포트 구조 (consulting-report 스킬 디자인 시스템 활용):
- **헤더**: 프로젝트명, 검증일, 대상 URL
- **요약 대시보드**: 검증 항목 N개 / 통과 X / 경고 Y / 실패 Z
- **항목별 상세 카드**: 각 트래커별로
  - 상태 뱃지 (GREEN/YELLOW/RED)
  - 발견된 ID (G-XXXXXXX 등)
  - fire된 이벤트 목록 + 응답 코드
  - 스크린샷 썸네일 (클릭하면 lightbox)
  - 네트워크 요청 raw payload (접을 수 있게)
- **이슈 리스트**: 발견된 모든 문제 + 권장 조치
- **클라이언트 응답 스크립트**: "여기 사진처럼 적용 완료되었습니다" 포함 텍스트

### Step 6 — 클라이언트 반박 방지 응답 스크립트 생성

리포트 마지막에 다음 형태 텍스트 자동 생성 (이모지 금지, 글로벌 규칙):

```
--
안녕하세요, 광고 픽셀 / GA4 설치 검증 완료되어 회신드립니다.

검증 도메인: https://<url>
검증 일시: 2026년 5월 22일 14:30

[1] Google Analytics 4
- Measurement ID: G-XXXXXXX
- page_view 이벤트 정상 발사 (응답 200)
- 검증 페이지: 홈 / 소개 / 가격 / 문의 (총 4개) 모두 정상
- 첨부: evidence/ga4-home.png, evidence/ga4-pricing.png

[2] Meta Pixel
- Pixel ID: 1234567890123456
- PageView 이벤트 정상 발사 (응답 200)
- Lead 이벤트는 폼 제출 시에만 발사되도록 정확히 설정됨
- 첨부: evidence/meta-home.png

[3] Google Tag Manager
- Container ID: GTM-XXXXXX
- 모든 페이지에 정상 로드됨

추가로 GA4 DebugView 및 Meta Events Manager Test Events 화면도
첨부드렸으니 함께 확인 부탁드립니다.

혹시 추가 검증이 필요한 페이지나 이벤트가 있으시면 말씀 주세요.
--
```

### Step 7 — 자동 수정 루프 (Auto-Fix Loop)

검증 → 실패 발견 → 수정 → 재검증을 **모든 항목이 GREEN이 되거나 escape 조건에 걸릴 때까지** 반복한다.

#### 루프 정책

```
최대 반복: 5회 (--max-loops 옵션으로 조정 가능, 기본 5)
타임아웃: 전체 30분 (--timeout 옵션)
변화 없음 감지: 직전 2회와 동일한 진단 결과 → escape (무한 루프 방지)
사용자 확인 트리거: 자동 수정 불가 영역 발견 시 즉시 사용자에게 보고
```

#### 자동 수정 가능 / 불가 분류

**자동 수정 가능 (LOOP)** — 코드/설정 파일 수정으로 해결:
| 진단 | 자동 수정 액션 |
|---|---|
| 플레이스홀더 ID 잔존 (`G-XXXXXXX`) | 사용자에게 실제 ID 1회 질문 → 모든 파일 일괄 치환 |
| `UA-` 패턴 잔존 | grep으로 위치 찾아 GA4 패턴으로 교체 (gtag config) |
| gtag + GTM 동시 설치 | 둘 중 하나 제거 (GTM 우선 유지, gtag 직접 호출 제거) |
| 일부 페이지에 스크립트 누락 | 공통 레이아웃(`app/layout.tsx`, `BaseLayout.astro` 등)으로 이동 |
| Consent Mode V2 신호 누락 | gtag('consent', 'default', {...}) 4신호 head 최상단 삽입 |
| PageView 중복 fire | fbq('init') 중복 호출 제거 |
| Lead가 페이지 로드 시 잘못 fire | 트리거를 button.click 이벤트로 이동 |
| GTM trigger가 DOM Ready | 코드 내 GTM dataLayer push 시점 조정 |
| 셀렉터 취약성 (data-testid 없음) | 해당 element에 data-testid 추가 |

**자동 수정 불가 (ESCAPE)** — 사용자/외부 시스템 필요:
| 진단 | 안내 액션 |
|---|---|
| GA4 Measurement ID 자체를 모름 | 사용자에게 "Analytics 관리자 → 데이터 스트림에서 G-... 가져와줘" |
| Meta Pixel ID 자체를 모름 | 사용자에게 "Events Manager → 데이터 소스에서 픽셀 ID 가져와줘" |
| GA4 DebugView 확인 필요 | 수동 가이드 + 스크린샷 요청 |
| Events Manager Test Events 확인 | 수동 가이드 + 스크린샷 요청 |
| Custom Dimension 등록 | "GA4 관리자 → 맞춤 정의에서 등록 필요" 안내 |
| Key Event(전환) 마크 | "GA4 관리자 → 이벤트에서 generate_lead 전환 토글" 안내 |
| CAPI 서버사이드 미연동 | "별도 작업 필요. 권한과 액세스 토큰 받아서 진행" 안내 |
| 광고 매니저 어트리뷰션 | "캠페인 보고서에서 확인 필요" 안내 |
| 도메인 verification | "Facebook Business Manager에서 처리" 안내 |

#### 루프 알고리즘 (의사 코드)

```ts
async function autoFixLoop(maxLoops = 5, timeoutMs = 30 * 60 * 1000) {
  const startTime = Date.now();
  const history: DiagnosisSnapshot[] = [];

  for (let loop = 1; loop <= maxLoops; loop++) {
    console.log(`\n=== Loop ${loop}/${maxLoops} ===`);

    if (Date.now() - startTime > timeoutMs) {
      report('TIMEOUT', '전체 타임아웃 30분 초과 — 사용자 확인 필요');
      break;
    }

    // 1. 검증 실행 (Step 3 그대로)
    const diagnosis = await runFullVerification();
    history.push(diagnosis);

    // 2. 모두 GREEN이면 완료
    if (diagnosis.allGreen()) {
      report('SUCCESS', `${loop}회 루프 만에 모든 검증 통과`);
      return;
    }

    // 3. 변화 없음 감지 (직전 2회와 동일)
    if (history.length >= 3) {
      const last3 = history.slice(-3);
      if (isSameDiagnosis(last3[0], last3[1]) && isSameDiagnosis(last3[1], last3[2])) {
        report('STUCK', '직전 2회 루프와 동일한 진단 결과 — 자동 수정 한계, 사용자 개입 필요');
        await dumpDiagnosis(diagnosis);
        break;
      }
    }

    // 4. 자동 수정 가능 항목 분리
    const fixable = diagnosis.issues.filter(i => i.category === 'AUTO_FIXABLE');
    const escape = diagnosis.issues.filter(i => i.category === 'ESCAPE');

    // 5. ESCAPE 항목이 있으면 즉시 사용자에게 보고
    if (escape.length > 0) {
      report('USER_INPUT_REQUIRED', escape);
      const userResponse = await askUser(escape);  // 1회 질문, 답 받아서 계속
      if (!userResponse) break;
      applyUserInput(userResponse);  // ID 등 사용자가 제공한 값 적용
    }

    // 6. 자동 수정 적용
    if (fixable.length === 0 && escape.length === 0) {
      report('UNKNOWN_FAILURE', '진단은 실패 표시인데 수정 액션이 없음 — 사용자 확인');
      break;
    }

    for (const issue of fixable) {
      console.log(`  → 수정 시도: ${issue.title}`);
      const fixResult = await applyAutoFix(issue);
      if (!fixResult.success) {
        console.log(`  ✗ 수정 실패: ${fixResult.reason}`);
      }
    }

    // 7. 빌드 & 배포 (코드 수정이 있었으면)
    if (fixable.some(f => f.requiresRebuild)) {
      console.log('  → 빌드 + 배포 진행');
      await runBuild();
      await runDeploy();
    }
  }

  // 루프 종료 — 최종 리포트 생성
  await generateFinalReport(history);
}
```

#### 루프 실행 중 사용자 보고 (1줄씩, 글로벌 규칙)

```
[Loop 1/5] 검증 시작 → 실패 4건 발견
  ✗ GA4: Measurement ID에 'G-XXXXXXX' 플레이스홀더 잔존
  ✗ Meta: PageView 중복 fire (2회)
  ✗ Consent Mode V2 신호 없음
  ⚠ /pricing 페이지에 GTM 누락 (ESCAPE 아님, 자동 수정 가능)

  사용자에게 필요한 정보:
  → 실제 GA4 Measurement ID 알려줘 (예: G-ABC1234567)

[사용자 응답 받음: G-ABC1234567]

[Loop 1/5] 자동 수정 적용 중
  → src/lib/analytics.ts: G-XXXXXXX → G-ABC1234567 치환
  → public/index.html: fbq('init') 중복 1개 제거
  → src/layouts/Base.astro: Consent Mode V2 default 신호 추가
  → src/pages/pricing.astro: GTM 스니펫 누락 → 공통 레이아웃으로 이동

[Loop 1/5] 빌드 + 배포 진행
  → pnpm build 성공
  → Vercel 배포 성공 (URL: ...)

[Loop 2/5] 재검증 시작 → 실패 1건 남음
  ✗ Meta: Lead 이벤트가 페이지 로드 시점에 잘못 fire

[Loop 2/5] 자동 수정 적용
  → src/components/ContactForm.tsx: fbq('Lead') 위치를 onSubmit 핸들러로 이동

[Loop 2/5] 빌드 + 배포 → 성공

[Loop 3/5] 재검증 시작 → 모든 검증 GREEN
✓ 완료. 총 3회 루프, 12분 22초 소요.
```

#### Escape 조건 상세

다음 중 하나 발생 시 루프 중단 → 사용자에게 명확히 보고:

1. **MAX_LOOPS 도달**: "5회 루프 후에도 N건 미해결. 마지막 진단 결과 첨부."
2. **TIMEOUT**: "30분 초과. 진행 상태 첨부."
3. **STUCK**: "직전 2회 루프와 동일한 결과 = 자동 수정 한계. 진단 + 시도한 액션 첨부."
4. **USER_INPUT_REQUIRED**: ID/토큰/외부 서비스 설정 필요 — 사용자 응답 대기.
5. **BUILD_FAILED**: 빌드 실패 → 수정 코드 자체에 문제. 변경 사항 revert 후 사용자 보고.
6. **DEPLOY_FAILED**: 배포 실패 (Vercel git author 등) → 별도 가이드로 안내.

#### 안전장치 (반드시 지킴)

- **자동 수정 전 git stash**: 매 루프 시작 시 `git stash push -m "verify-pixels-loop-N"` → 롤백 가능
- **빌드 실패 시 즉시 revert**: `pnpm build` 실패 → `git stash pop` 또는 `git checkout` 으로 직전 상태로
- **프로덕션 직배포 금지**: 코드 수정 → 빌드 → preview 배포 → 검증 → 통과 시에만 prod (또는 사용자 컨펌)
- **3회 같은 파일 수정 감지 시 escape**: 동일 파일을 3번 이상 수정해도 통과 못하면 사용자 개입
- **DROP/DELETE SQL 자동 실행 금지** (글로벌 규칙)
- **외부 서비스 API 호출 자동화 금지**: GA4/Meta 대시보드 변경은 사용자만

#### 사용자 옵션

```
/verify-pixels --loop                  # 자동 수정 루프 활성 (기본은 비활성, 진단만)
/verify-pixels --loop --max-loops 3    # 최대 3회로 제한
/verify-pixels --loop --no-deploy      # 빌드만, 배포는 사용자가
/verify-pixels --loop --interactive    # 매 수정마다 사용자 컨펌 (안전 모드)
/verify-pixels --loop --dry-run        # 진단 + 수정 액션 출력만, 실제 변경 X
```

#### 최종 리포트에 추가 섹션

루프가 끝난 후 HTML 리포트에 다음 섹션 추가:

```html
<section class="loop-history">
  <h2>자동 수정 루프 히스토리</h2>
  <div class="timeline">
    <div class="loop-step">
      <div class="loop-num">Loop 1</div>
      <div class="loop-found">발견: 4건</div>
      <div class="loop-fixed">수정: 3건 자동 + 1건 사용자 응답</div>
      <div class="loop-build">빌드/배포: 성공 (1분 23초)</div>
    </div>
    <!-- ... -->
  </div>
  <div class="loop-summary">
    총 N회 루프, X분 Y초 소요. 최종 상태: 전체 통과
  </div>
</section>
```

---

## 정직성 원칙

### "통과"라는 표현 금지, 정확하게 표기

| 정확한 보고 | 부정확한 보고 |
|---|---|
| "page_view 1회 정상 fire, 200 응답" | "GA4 정상" |
| "Pixel Helper 미검증 (확장 자동화 불가) — 수동 확인 가이드 첨부" | "Meta Pixel 완벽 설치" |
| "클라이언트 사이드만 검증됨. CAPI 서버사이드는 별도 확인 필요" | "트래킹 완료" |
| "EEA 대상이면 Consent Mode V2 신호 추가 필요" | (누락) |

### Pixel Helper "Green" 의미 정확히 표기

리포트에 다음 문구 포함:
> Pixel Helper의 녹색 체크는 "브라우저에서 요청이 떠났음"을 의미하며, "Meta 서버가 이벤트를 수신/처리/어트리뷰션에 사용함"을 보장하지 않습니다. 서버 수신 검증은 Events Manager Test Events에서, 어트리뷰션 적용은 캠페인 보고서에서 별도 확인이 필요합니다.

### CAPI / 서버사이드 미검증 명시

이 스킬은 **클라이언트 사이드만 검증**. CAPI(Conversions API), 서버사이드 GTM, Measurement Protocol은 별도 도구 필요. 리포트 푸터에 명시.

---

## 흔한 함정 (체크리스트로 변환)

검증 시 다음을 자동 확인:

- [ ] Measurement ID / Pixel ID 가 플레이스홀더가 아님 (`XXXXXXX`, `YOUR_ID` 등 X)
- [ ] GA4 ID가 `G-`로 시작 (`UA-`면 Universal Analytics 잔존, RED)
- [ ] PageView 정확히 1회 fire (2회 이상이면 중복 설치)
- [ ] gtag + GTM 동시 존재하지 않음 (있으면 중복 위험 경고)
- [ ] 모든 핵심 페이지(홈/가격/소개/문의)에 트래커 로드
- [ ] 응답 코드 200/204
- [ ] Lead/Purchase 이벤트가 페이지 로드 시점에 잘못 fire되지 않음
- [ ] Consent Mode V2 default 신호 존재 (EEA 타겟이면)
- [ ] event_id 가 Pixel 이벤트에 포함됨 (CAPI 연동 시)
- [ ] 콘솔 에러 0개 (`fbq`, `gtag` 관련 에러 없음)
- [ ] GTM trigger가 "All Pages" 또는 "Page View" 사용 (DOM Ready / Window Loaded X)
- [ ] purchase 이벤트에 transaction_id, value, currency 모두 포함
- [ ] generate_lead 이벤트가 GA4에서 key event(전환)로 마크됨 (수동 확인 안내)

---

## Page Object 패턴 (재사용)

```ts
// tests/tracking/lib/tracker-recorder.ts
export class TrackerRecorder {
  hits: TrackerHit[] = [];

  attach(page: Page) {
    page.on('request', (req) => { /* ... */ });
    page.on('response', (res) => { /* ... */ });
  }

  byPlatform(platform: string) { return this.hits.filter(h => h.platform === platform); }
  byEvent(name: string) { return this.hits.filter(h => h.params.en === name || h.params.ev === name); }
  hasNoDuplicates(platform: string, event: string) { /* ... */ }
  saveEvidence(dir: string, label: string) { /* json + png */ }
}
```

---

## 금지 사항

- ❌ "트래킹 완벽 설치" / "100% 정상" 같은 단정적 표현 (CAPI/서버 미검증 상태에서)
- ❌ Pixel Helper 그린만 보고 OK 판정
- ❌ 홈 1페이지만 검증하고 "전체 사이트 OK"
- ❌ Consent Mode V2 누락된 EEA 사이트를 PASS 처리
- ❌ 스크린샷 없이 텍스트로만 보고 (클라이언트 반박 방지가 핵심 목적)
- ❌ 플레이스홀더 ID(`G-XXXXXXX` 등)를 발견하고도 정상 처리
- ❌ 외부 URL 검증 시 사용자 동의 없이 무한 페이지 크롤링
- ❌ 자동 수정 루프에서 `--max-loops` 초과 강행
- ❌ 빌드 실패한 코드 변경을 그대로 둔 채 다음 루프 진행 (반드시 revert)
- ❌ 자동 수정 루프에서 GA4/Meta 대시보드 등 외부 서비스 변경 시도
- ❌ 사용자 응답 대기 중 임의로 다음 루프 진행
- ❌ STUCK 상태(같은 진단 3회)에서 같은 수정 액션 반복

---

## 사용자 시나리오별 출력

### 시나리오 1: 클라이언트가 "픽셀 안 깔린 것 같아요" 라고 함

→ `/verify-pixels --url <클라이언트URL> --quick`
→ 결과: HTML 리포트 + 스크린샷 4~5장
→ 응답 스크립트:
```
--
첨부 사진처럼 Meta Pixel(ID: ...)과 GA4(ID: G-...)가 정상 적용되어
PageView 이벤트가 200 응답으로 발사되고 있음을 확인했습니다.

evidence/meta-home.png — 네트워크 탭에서 facebook.com/tr 요청 200 응답
evidence/ga4-home.png — google-analytics.com/g/collect 요청 200 응답
evidence/source-code.png — HTML 소스에 픽셀 코드 정상 포함

혹시 광고 매니저 화면에서 이벤트가 안 보이신다면 Aggregated Event
Measurement 우선순위 설정 또는 Consent Mode 영향일 수 있으니
추가 확인 도와드리겠습니다.
--
```

### 시나리오 2: 신규 배포 직후 자체 검증

→ `/verify-pixels`
→ 결과: 전체 페이지 × 전체 트래커 매트릭스
→ 실패 항목만 추출해서 수정 후 재실행

### 시나리오 3: 특정 이벤트(Lead, Purchase)만 검증

→ `/verify-pixels --event Lead --flow "/contact → 폼 작성 → 제출"`
→ 결과: 폼 제출 시점 네트워크 캡처 + payload 검증

---

## 출력 톤 (글로벌 규칙 준수)

- 한국어, 반말, 간결
- 클라이언트 응답 스크립트 부분만 정중한 비즈니스 한국어 (이모지 금지)
- 진행 중 업데이트 1줄씩
- 작업 완료 후 ntfy 알림 (Title: "트래킹 검증 완료 — G-XXX / Pixel XXX 정상")
