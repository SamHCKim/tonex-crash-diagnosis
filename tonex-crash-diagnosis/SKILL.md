---
name: tonex-crash-diagnosis
description: Windows 데스크톱 앱이 실행 즉시 크래시할 때(ucrtbase.dll, 0xc0000409, 0xc0000005 등) 크래시 덤프·이벤트 로그·보안 설정을 한 번에 수집·분석해 원인 순위와 해결책을 한 번에 보고한다.
---

# Windows 앱 실행 크래시 원스톱 진단

## 목적
사용자와 여러 번 주고받지 않고, **사용자 행동 1회(폴더 접근 승인 + PowerShell 1회 붙여넣기)**만으로 아래 원인 후보 전체를 검증한다. 결과로 **원인 순위, 해결책, 되돌리는 법, 검증 방법**을 한 번에 보고한다. Windows 설정 변경이나 재설치 같은 실제 조치는 사용자가 직접 한다.

## 사용 환경
- **Claude Cowork + 사용자 PC 연결 시:** `device_request_folder_access`, `device_bash`로 진행한다.
- **Windows에서 직접 실행되는 에이전트(Claude Code 등):** 같은 순서로 진행하되, PowerShell 수집 명령을 직접 실행하고 덤프 분석 스크립트는 Python으로 로컬에서 실행한다.

## 절대 규칙
- 진단 단계는 **읽기 전용**이다. 파일 이름 변경·삭제·DB 수정은 백업한 뒤 사용자 확인을 받고 나서만 한다.
- 시스템 설정(UTF-8 옵션, 보안 프로그램, 드라이버, 레지스트리)은 **사용자가 직접** 바꾸도록 안내만 한다.
- Cowork의 computer use로는 터미널(PowerShell)에 입력할 수 없다(클릭 전용). 명령은 사용자가 붙여넣게 한다.
- 확인된 사실과 추론을 구분해 보고한다. 확인하지 못한 것은 "미확인"으로 쓴다.

## 진행 순서 (총 2턴 목표)

### 턴 1 — 아래를 한 메시지에서 동시에 처리
1. `get_device_info`로 앱 설치 폴더와 사용자 데이터 폴더 위치를 파악한다. 폴더 이름은 `device_list_dir`의 이름 목록으로 확인한다.
2. `device_request_folder_access`로 아래 폴더를 **한 번에** 요청한다. 모두 읽기 목적이다.
   - `~/AppData/Local/CrashDumps`
   - 앱 설치 폴더 (예: `C:\Program Files\<Vendor>\<App>`)
   - 문서 안의 벤더 폴더 (예: `~/Documents/<Vendor>`)
   - `~/AppData/Roaming/<Vendor>` (있으면)
3. 사용자에게 아래 **PowerShell 수집 스크립트**를 전달한다. `$app` 값만 앱 이름으로 바꾼다. 안내 문구:
   - "관리자 권한 PowerShell에 붙여넣고 Enter를 누르세요. `>>`가 보이면 Enter를 한 번 더 누르면 됩니다. 줄이 꺾여 보이는 것은 화면에서만 접힌 것이라 정상입니다."
4. 사용자가 실행하는 동안 덤프 분석(아래 스크립트)과 폴더 구조 분석을 먼저 끝낸다.

### 턴 2 — `app_diag.txt`를 읽고 종합해 최종 보고서를 한 번에 전달한다

## PowerShell 수집 스크립트
```powershell
$app='TONEX'   # 앱 이름(또는 exe 이름 일부)
$o="$env:LOCALAPPDATA\CrashDumps\app_diag.txt"
New-Item -ItemType Directory -Force (Split-Path $o) | Out-Null
& {
"== SYSTEM"; (Get-CimInstance Win32_OperatingSystem).Caption + ' ' + [Environment]::OSVersion.Version; "UserProfile=$env:USERPROFILE"; "Documents=" + [Environment]::GetFolderPath('MyDocuments'); "ACP=" + (gp HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage).ACP; Get-WinSystemLocale | ft -auto
"== APP EVENTS"; Get-WinEvent -FilterHashtable @{LogName='Application';Id=1000,1001,1026;StartTime=(Get-Date).AddDays(-7)} -EA SilentlyContinue | ? Message -match $app | select -First 20 | fl TimeCreated,Id,Message
"== DEFENDER EVENTS"; Get-WinEvent -LogName 'Microsoft-Windows-Windows Defender/Operational' -MaxEvents 1000 -EA SilentlyContinue | ? {$_.Id -in 1116,1117,1121,1122,1123,1124} | select -First 30 | fl TimeCreated,Id,Message
"== DEFENDER PREFS"; try { Get-MpPreference -EA Stop | fl EnableControlledFolderAccess,ControlledFolderAccessAllowedApplications,AttackSurfaceReductionRules_Ids,AttackSurfaceReductionRules_Actions,ExclusionPath } catch { "Get-MpPreference failed: $($_.FullyQualifiedErrorId) / $($_.Exception.Message)" }
"== AV PRODUCTS"; Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntiVirusProduct -EA SilentlyContinue | ft displayName,productState -auto
"== SECURITY SOFTWARE"; gp HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*,HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*,HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\* -EA SilentlyContinue | ? DisplayName -match 'AhnLab|V3|ALYac|ESTsoft|MarkAny|nProtect|INCA|TouchEn|Raon|Wizvera|Veraport|AnySign|IPinside|Kaspersky|Avast|AVG|Norton|McAfee|Bitdefender|ESET|Sophos|Trend' | ft DisplayName,DisplayVersion -auto
"== VC++ RUNTIME"; gp HKLM:\SOFTWARE\Microsoft\VisualStudio\14.0\VC\Runtimes\x64 -EA SilentlyContinue | fl Version
"== GPU"; Get-CimInstance Win32_VideoController | ft Name,DriverVersion,DriverDate -auto
"== AUDIO"; Get-CimInstance Win32_SoundDevice | ft Name,Manufacturer,Status -auto
"== APP FOLDERS"; foreach($p in "$env:APPDATA","$env:LOCALAPPDATA",[Environment]::GetFolderPath('MyDocuments'),"$env:ProgramFiles","${env:ProgramFiles(x86)}"){ gci $p -Directory -Recurse -Depth 2 -EA SilentlyContinue | ? FullName -match $app | select FullName,LastWriteTime,@{n='Files';e={(gci $_.FullName -File -EA SilentlyContinue).Count}} | ft -auto }
"== LOCALDUMPS"; gp 'HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps' -EA SilentlyContinue | fl
"== CRASHDUMPS"; gci "$env:LOCALAPPDATA\CrashDumps" -EA SilentlyContinue | sort LastWriteTime -desc | select -First 15 Name,Length,LastWriteTime | ft -auto
} *>&1 | Out-File $o -Encoding utf8 -Width 300
"완료: $o"
```
결과 파일은 이미 연결된 CrashDumps 폴더에 저장되므로 `device_bash`로 바로 읽는다.

## 덤프 분석 스크립트
- 준비: `pip install --user -q minidump pefile`
- 실행: `python dump_triage.py "<CrashDumps 경로>" "<앱 설치 폴더 경로>"`
- Windows에서는 `python`(없으면 `py`)을 쓴다. Windows의 `python3`는 Microsoft Store 연결용 빈 명령일 수 있다. Cowork 같은 Linux 환경에서는 `python3`를 쓴다.
- Cowork에서는 경로가 `$HOME/mnt/CrashDumps`, `$HOME/mnt/<설치폴더 마운트명>` 형태다.

```python
# dump_triage.py
import sys,os,struct,glob,logging,collections
logging.disable(logging.CRITICAL)
sys.stdout.reconfigure(errors='backslashreplace')  # 한글을 못 쓰는 콘솔(영어·일본어 등)에서도 중단되지 않게 한다
from minidump.minidumpfile import MinidumpFile
try: import pefile
except ImportError: pefile=None
FASTFAIL={2:'STACK_COOKIE(GS 버퍼 오버런)',7:'FATAL_APP_EXIT(abort: 처리 안 된 C++ 예외)'}
WINERR={2:'파일 없음',3:'경로 없음',5:'접근 거부',32:'공유 위반(다른 프로세스 잠금)',123:'잘못된 파일명',206:'경로 너무 김',1113:'유니코드→코드페이지 변환 실패'}
SEC=['ahnlab','v3','alyac','estsoft','markany','imgsf','nprotect','npk','inca','touchen','raon','wizvera','veraport','anysign','ipinside','kaspersky','avast','avg','norton','symantec','mcafee','bitdefender','eset','sophos','trend','crowdstrike','sentinel']
dd=sys.argv[1]; appdir=sys.argv[2] if len(sys.argv)>2 else None
pecache={}
def pe_read(modpath,rva,n):
    nm=modpath.split('\\')[-1]
    if nm not in pecache:
        p=os.path.join(appdir,nm) if appdir else ''
        pecache[nm]=pefile.PE(p,fast_load=True) if (pefile and os.path.exists(p)) else None
    pe=pecache[nm]
    return pe.get_data(rva,n) if pe else None
agg=collections.Counter(); third=collections.Counter()
for d in sorted(glob.glob(os.path.join(dd,'*.dmp')),key=os.path.getmtime,reverse=True):
    try: mf=MinidumpFile.parse(d)
    except Exception as e: print(os.path.basename(d),'파싱 실패',e); continue
    er=mf.exception.exception_records[0]; r=er.ExceptionRecord
    code=getattr(r,'ExceptionCode_raw',None)
    if code is None:
        c=r.ExceptionCode; code=c if isinstance(c,int) else getattr(c,'value',0)
    params=list(r.ExceptionInformation[:r.NumberParameters])
    mods=[(m.baseaddress,m.endaddress,m.name) for m in mf.modules.modules]
    main=mods[0][2] if mods else ''
    appd=main.rsplit('\\',1)[0].lower()
    fm=next((f"{n.split(chr(92))[-1]}+{hex(r.ExceptionAddress-b)}" for b,e,n in mods if b<=r.ExceptionAddress<e),'?')
    for b,e,n in mods:
        l=n.lower()
        if not l.startswith('c:\\windows\\') and not l.startswith(appd):
            third[n]+=1
    cxx=''
    th=[t for t in mf.threads.threads if t.ThreadId==er.ThreadId]
    if th:
        th=th[0]
        with open(d,'rb') as f:
            f.seek(th.Stack.MemoryLocation.Rva); data=f.read(th.Stack.MemoryLocation.DataSize)
        for i in range(0,len(data)-64,4):
            if struct.unpack_from('<I',data,i)[0]==0xE06D7363 and struct.unpack_from('<I',data,i+24)[0]==4:
                magic,obj,ti,img=struct.unpack_from('<4Q',data,i+32)
                mp=next((n for b,e,n in mods if b<=img<e),None)
                types=[]
                try:
                    raw=pe_read(mp,ti-img,16) if mp else None
                    if raw:
                        cta=struct.unpack('<4i',raw)[3]; n=struct.unpack('<i',pe_read(mp,cta,4))[0]
                        for k in range(n):
                            ct=struct.unpack('<i',pe_read(mp,cta+4+4*k,4))[0]
                            pt=struct.unpack('<2i',pe_read(mp,ct,8))[1]
                            types.append(pe_read(mp,pt+16,200).split(b'\0')[0].decode(errors='replace'))
                except Exception as ex: types.append(f'(타입 해석 실패 {ex})')
                cxx='C++ '+(types[0] if types else '?')
                try:
                    o=mf.get_reader().read(obj,40)
                    if types and 'system_error' in types[0]:
                        ec=struct.unpack_from('<i',o,24)[0]; cxx+=f' 오류코드 {ec}({WINERR.get(ec,"미등록")})'
                except Exception: pass
                break
    desc=hex(code & 0xffffffff)
    if (code & 0xffffffff)==0xC0000409 and params: desc+=f' param{params[0]}={FASTFAIL.get(params[0],"?")}'
    print(f"{os.path.basename(d)} | {desc} | 오류모듈 {fm} | {cxx}"); agg[(desc,fm,cxx)]+=1
print('\n== 패턴 집계')
for k,v in agg.most_common(): print(v,'x',k)
print('\n== Windows·앱 폴더 밖 모듈(주입 DLL 후보)')
for n,v in third.most_common():
    flag=' ← 보안 프로그램 의심' if any(s in n.lower() for s in SEC) else ''
    print(v,n,flag)
```

스크립트가 실패하면 대안으로 확인한다.
- `strings -n 6 <덤프>.dmp | grep '\.?AV'`: 예외 타입 후보
- `strings -el <덤프>.dmp`: 경로 문자열

## 원인 후보와 판정 기준
| ID | 원인 후보 | 강한 신호 (사실로 확인 가능) | 해결책 (사용자 직접) |
|---|---|---|---|
| H1 | **비영문 경로·코드페이지 불일치** | 덤프에 `system_error 1113`, 사용자 경로에 한글 등 비ASCII 문자, ACP≠65001 | 제어판 → 국가 또는 지역 → 관리자 옵션 → 시스템 로캘 변경 → "Beta: 세계 언어 지원을 위해 Unicode UTF-8 사용" 체크 → 재부팅. 대안: 영문 이름 로컬 계정. 벤더에 신고 |
| H2 | **사용자 데이터·설정 손상 또는 비호환** | 데이터 폴더가 없을 때만 실행됨, 오류코드 2·3·123, 이전 버전 파일 혼재 | 데이터 폴더를 백업한 뒤 이름을 바꾸고 실행 테스트. 문제 파일 범위 좁히기 |
| H3 | **보안 프로그램 주입·차단** | 주입 DLL 후보에 보안 벤더 모듈, Defender 1116~1124 이벤트, 오류코드 5·32, 다른 백신 활성 | 해당 프로그램에 앱 폴더를 예외 등록하거나 잠시 중지한 뒤 테스트. 중지했다면 테스트 후 반드시 다시 켜기 |
| H4 | **설치 손상·런타임 누락** | 0xc0000135 또는 0xc000007b, 앱 DLL에서 크래시, VC++ 런타임 없음 | VC++ 재배포 패키지 설치, 클린 재설치 |
| H5 | **그래픽 드라이버** | 0xc0000005 + 오류 모듈이 igd*, nvoglv*, atio*, d3d* | 드라이버 업데이트 또는 롤백 |
| H6 | **오디오·ASIO 드라이버** | 오류 모듈이 ASIO나 오디오 드라이버 DLL | 드라이버 업데이트, 장치 변경 |
| H7 | **앱 빌드 자체 결함** | 위 신호가 없음, 최근 빌드, 환경 요인 모두 배제 | 이전 버전 설치, 벤더에 덤프를 첨부해 신고 |

판정 원칙:
- `0xc0000409 param7`은 앱이 스스로 abort한 것이다. **반드시 C++ 예외 타입과 오류코드를 추출**해 판정 근거로 쓴다. ucrtbase.dll은 종료가 기록된 위치일 뿐 원인이 아니다.
- 오류코드가 나오면 그 코드가 가리키는 후보를 1순위로 두고, 나머지는 반대 근거로 지운다.
- 데이터 파일(DB 등)의 수정 시각이 크래시 이후에도 바뀌지 않았다면 데이터를 열기 전 단계에서 종료한 것이다. 이때는 데이터 내용 가설(H2 일부)이 약해진다.

## 함정 (실제 사례에서 얻은 교훈)
- **첫 실행 성공에 속지 말 것:** 데이터 폴더를 지우면 한 번만 실행되고 다음부터 다시 크래시할 수 있다. 검증은 항상 "종료 후 두 번 이상 다시 실행"으로 한다.
- **사례 (TONEX 2.1.1, 2026-10):**
  - 증상: 모든 덤프가 `std::system_error 1113`, 사용자 폴더 `C:\Users\<한글 이름>`, ACP=949
  - 해결: UTF-8 옵션으로 해결됨
  - 효과 없었던 시도: 라이브러리 DB 안의 경로 수정, 데이터 폴더 이름 변경
  - 다른 한국 사용자에게 드문 이유(추론): Microsoft 계정으로 설정하면 사용자 폴더가 영문으로 만들어짐
- CrashDumps에는 최근 10개 정도만 남는다. 오래된 덤프는 사라질 수 있다.
- 마운트된 폴더의 대용량 복사는 `device_bash` 시간 제한(180초)을 넘길 수 있다. 백업은 가능하면 이름 변경(`mv -n`)으로 하고, 복사가 필요하면 완료 여부를 다시 확인한다.
- `Get-MpPreference` 오류 0x800106ba는 Defender 서비스가 꺼져 있다는 뜻이다. 다른 백신이 활성일 가능성을 보여준다.
- 덤프가 하나도 없으면 LocalDumps 레지스트리 설정이 필요하다. 레지스트리 변경이므로 사용자 확인 후 안내만 한다.

## 최종 보고서 형식 (한 번에 전달)
1. **결론 한 줄:** 가장 유력한 원인 + 해결책
2. **결정적 증거** (사실): 예외 코드, 오류코드, 이벤트, 환경값
3. **원인 후보 순위 표:** 원인 후보 | 근거 | 반대 근거 | 확신도 (높음·중간·낮음, 추론임을 표시)
4. **해결 단계** (사용자가 직접): 순서대로, 각 단계의 부작용과 되돌리는 법 포함
5. **검증 방법:** 재부팅 후 앱을 두 번 이상 실행하고, 관련 앱(에디터 등)도 확인
6. **안 될 경우 다음 수:** 2순위 후보의 최소 테스트 1개
7. **출처:** 분석한 로컬 파일과 공식 문서 링크
