# tonex-crash-diagnosis

Windows 데스크톱 앱이 **실행하자마자 꺼질 때**, 원인을 한 번에 짚어주는 Claude 스킬입니다.

크래시 덤프, 이벤트 로그, 보안 프로그램, 시스템 로캘 정보를 한꺼번에 모아 분석합니다. 그 결과로 **원인 순위, 해결책, 되돌리는 방법, 검증 방법**을 한 번에 알려줍니다.

> A Claude skill that diagnoses Windows desktop apps crashing on launch (e.g. `ucrtbase.dll`, `0xc0000409`) by analyzing crash dumps, event logs and security settings in one pass. Documentation is in Korean.

IK Multimedia와 무관한 **비공식** 도구입니다. TONEX와 AmpliTube는 IK Multimedia의 상표입니다. (Unofficial; not affiliated with IK Multimedia.)

## 이런 증상에 씁니다

- 앱이 실행 직후 아무 메시지 없이 종료됨
- 이벤트 로그에 `ucrtbase.dll`, `0xc0000409`, `0xc0000005` 같은 오류가 기록됨
- 같은 회사의 다른 앱은 정상인데 특정 앱만 실행되지 않음
- 재설치해도 증상이 그대로임

## 작동 방식

사용자가 할 일은 **두 가지**입니다.

1. 폴더 접근 요청을 한 번 승인합니다 (크래시 덤프, 앱 설치 폴더, 사용자 데이터 폴더).
2. Claude가 준 PowerShell 명령을 관리자 권한 PowerShell에 한 번 붙여넣습니다.

Claude는 그동안 크래시 덤프를 분석합니다. 앱이 어떤 예외로 종료됐는지와 Windows 오류 코드를 추출하고, PowerShell 수집 결과와 합쳐 최종 보고서를 만듭니다.

진단은 **읽기 전용**입니다. 파일 변경은 백업한 뒤 사용자 확인을 받고 나서만 하고, Windows 설정 변경은 사용자가 직접 합니다.

수집 결과는 `%LOCALAPPDATA%\CrashDumps\app_diag.txt`에 저장됩니다. 사용자 폴더 경로와 보안 프로그램 정보가 들어 있으니 진단이 끝나면 지우세요.

## 진단하는 원인 후보

| ID | 원인 후보 | 대표 신호 |
|---|---|---|
| H1 | 비영문(한글 등) 경로와 시스템 코드페이지 불일치 | 덤프에 `std::system_error` 오류 1113 |
| H2 | 사용자 데이터·설정 손상 또는 비호환 | 데이터 폴더가 없을 때만 실행됨 |
| H3 | 보안 프로그램 주입·차단 | 주입 DLL, Defender 차단 이벤트, 오류 5·32 |
| H4 | 설치 손상·런타임 누락 | `0xc0000135`, `0xc000007b` |
| H5 | 그래픽 드라이버 | 드라이버 DLL에서 `0xc0000005` |
| H6 | 오디오·ASIO 드라이버 | 오디오 드라이버 DLL에서 크래시 |
| H7 | 앱 빌드 자체 결함 | 환경 요인이 모두 배제됨 |

## 설치

### Claude 앱 (Claude.ai, Cowork)
1. 이 저장소의 `tonex-crash-diagnosis` 폴더를 zip으로 압축합니다.
2. Claude 설정의 스킬(Skills) 메뉴에서 zip 파일을 업로드합니다.

Cowork에서 사용자 PC 파일에 접근하려면 Claude 데스크톱 앱에서 PC가 연결돼 있어야 합니다.

### Claude Code

**방법 1. PowerShell에 붙여넣기 (권장)**

아래 한 줄을 PowerShell에 붙여넣고 Enter를 누릅니다. 스킬 파일(`SKILL.md`) 하나만 내려받아 스킬 폴더에 저장합니다.

```powershell
$d="$env:USERPROFILE\.claude\skills\tonex-crash-diagnosis"; New-Item -ItemType Directory -Force $d | Out-Null; Invoke-WebRequest "https://raw.githubusercontent.com/SamHCKim/tonex-crash-diagnosis/main/tonex-crash-diagnosis/SKILL.md" -OutFile "$d\SKILL.md" -UseBasicParsing
```

**방법 2. Claude Code에 요청하기**

Claude Code 대화창에 아래 문장을 붙여넣습니다.

```
https://github.com/SamHCKim/tonex-crash-diagnosis 의 tonex-crash-diagnosis 폴더를 내 ~/.claude/skills/ 에 설치해줘.
```

**방법 3. 직접 복사**

이 저장소의 `tonex-crash-diagnosis` 폴더를 `~/.claude/skills/`에 복사합니다.

설치가 끝나면 아래 위치에 파일이 있어야 합니다. 스킬이 보이지 않으면 Claude Code를 다시 시작하세요.

```
~/.claude/skills/tonex-crash-diagnosis/SKILL.md
```

업데이트할 때는 같은 방법을 다시 실행하면 새 파일로 덮어씁니다.

## 사용 예

```
Windows에서 TONEX 앱이 실행하자마자 꺼져. 이벤트 로그엔 ucrtbase.dll, 0xc0000409라고 떠. 원인 진단해줘.
```

## 실제 해결 사례

**TONEX 2.1.1 (IK Multimedia), 2026-10**

- **증상:** TONEX, TONEX Editor, Board Editor가 모두 실행 직후 종료됨 (`ucrtbase.dll`, `0xc0000409`). 같은 회사의 AmpliTube 5는 정상
- **원인:** 덤프 10개 모두에서 `std::system_error` 오류 1113("유니코드 문자를 대상 코드페이지로 변환할 수 없음")이 나옴. Windows 사용자 폴더 이름이 한글이고 시스템 코드페이지는 949(한국어)였음
- **해결:** 제어판 → 국가 또는 지역 → 관리자 옵션 → 시스템 로캘 변경 → "Beta: 세계 언어 지원을 위해 Unicode UTF-8 사용" 체크 → 재부팅
- **효과 없었던 시도:** 라이브러리 폴더 이름 변경, 라이브러리 DB 안의 경로 수정
- **주의:** UTF-8 옵션은 베타 기능이라 일부 오래된 프로그램에서 글자가 깨질 수 있습니다. 문제가 생기면 체크를 해제하고 재부팅하면 원래대로 돌아갑니다.

## 요구 사항

- 덤프 분석: Python 3, `minidump`, `pefile` 패키지 (스킬 실행 중 자동 설치를 시도합니다)
- 크래시 덤프가 `%LOCALAPPDATA%\CrashDumps`에 생성돼 있어야 정확도가 높습니다

## 한계

- 덤프의 디버그 심볼이 없으면 정확한 함수 위치는 알 수 없습니다. 예외 종류와 오류 코드로 판정합니다.
- 원인 순위의 일부는 추론입니다. 해결 후 반드시 **앱을 두 번 이상 다시 실행**해서 확인하세요.

## 기여

새로운 크래시 사례나 원인 후보는 Issue나 Pull Request로 보내주세요. 증상, 덤프 분석 결과, 해결 방법을 함께 적어 주시면 판정 기준에 반영하겠습니다.

진단 결과를 공유할 때는 사용자 이름과 폴더 경로를 가려 주세요. 크래시 덤프(`.dmp`) 원본과 `app_diag.txt` 원본은 개인정보가 들어 있으니 올리지 마세요.

## 라이선스
MIT License. `LICENSE` 파일 참고.
