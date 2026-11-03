# Freal World Manipulation

PwnSec에서 대회 중 ROP로 해결한 문제를, 대회 후 House of Apple 2 FSOP로 다시 푼 기록입니다. 문제는 제가 제작한 것이 아니며, 이 저장소에는 당시 분석과 후속 풀이 코드를 정리했습니다.

## 결과와 자료 상태

- 대회 중: PIE·libc·environ 주소를 누출하고 스택 반환 경로에 ROP를 구성해 해결했습니다.
- 대회 후: 같은 읽기·쓰기 조건에서 fake FILE과 wide vtable을 구성해 셸을 얻었습니다.
- 2026년 11월 3일: 문제 ZIP을 다시 확보해 저장소에 추가했습니다. 이번 업로드 과정에서 바이너리 실행이나 풀이 재현은 하지 않았습니다.

공개한 코드는 대회 후 다시 정리한 FSOP 풀이입니다. 대회 중 사용한 ROP 코드가 이 저장소에 포함된 것은 아닙니다.

## 파일

| 경로 | 내용 |
| --- | --- |
| `exploit_fsop.py` | 대회 후 작성한 FSOP 풀이. 기본 접속 대상만 로컬로 변경 |
| `challenge/frealworld_WR.zip` | 다운로드 폴더에서 복구한 ZIP 원본 |
| `challenge/src/freal` | ZIP에서 추출한 문제 실행 파일 |
| `challenge/Dockerfile` | ZIP에 포함된 Ubuntu 22.04 기반 실행 환경 |
| `challenge/flag.txt` | 원본의 테스트용 `flag{fake_flag}` |
| `SHA256SUMS` | 업로드 자료의 SHA-256 |

ZIP에는 `src/freal.i64`도 포함되어 있습니다. 이는 IDA 분석 데이터베이스이며 C/C++ 원본 소스가 아닙니다. 복구한 묶음에는 C/C++ 소스 파일이 없습니다. 문제와 배포 파일의 권리는 원저작자에게 있습니다.

## 처음 막힌 부분

음수 인덱스를 통한 OOB 접근을 찾았지만, 포인터를 두 번 역참조하기 때문에 바로 libc를 읽을 수 없었습니다. GDB에서 bss 주변 포인터를 조사한 뒤 바이너리 내부의 순환 참조를 이용해 PIE 기준 주소를 구했습니다.

## OOB 범위 확장

각각 유한한 double인 `1e308`과 `2`를 곱하면 결과가 무한대가 됩니다. 하지만 이 입력만으로는 `calibrated()` 검사를 통과하지 못했습니다. 먼저 0x1fffffff바이트를 초과하는 메모리를 요청해야 했고, 이후 곱셈 경로에서 큰 크기 값을 만들 수 있었습니다.

후속 코드에서는 0x800000바이트 요청을 64회 사용했습니다. 요청 크기의 합계는 512 MiB이며, 실제 메모리 사용량을 측정한 수치는 아닙니다.

## libc 누출과 쓰기 주소 찾기

stdout 관련 포인터로 여러 청크를 채우고, 확장된 OOB 읽기로 bss 뒤쪽을 탐색했습니다. stdout 주소를 찾은 뒤 libc 기준 주소를 계산했습니다.

채운 영역의 경계를 좁혀 쓰기 가능한 위치를 찾고, 포인터 슬롯을 바꾸는 동작과 실제 쓰기를 분리해 `write_mem()`을 구성했습니다.

## 대회 후 FSOP

쓰기 가능한 영역에 fake `_IO_FILE`, `_IO_wide_data`, wide vtable을 구성했습니다. `_IO_wfile_jumps`와 `doallocate` 경로를 이용해 `system("sh")`로 이어지도록 만들고, `_IO_list_all`을 바꾼 뒤 정상 종료 시 stream flush를 트리거했습니다.

코드는 PIE 누출, 크기 제한 해제, heap spray, libc 누출, 쓰기 영역 탐색, fake FILE 구성, 종료 트리거의 일곱 단계로 나눴습니다.

## 실행 환경 참고

원본 Dockerfile은 서비스 포트 9999를 사용합니다. 풀이 코드는 Python과 pwntools, 대상 환경과 일치하는 `libc.so.6`가 필요합니다. libc는 복구한 ZIP에 포함되어 있지 않습니다.

```sh
python3 -m pip install -r requirements.txt
LIBC=/path/to/matching/libc.so.6 python3 exploit_fsop.py
```

기본 대상은 `127.0.0.1:9999`입니다. `HOST`, `PORT`, `LIBC` 환경 변수로 접속 대상과 libc 경로를 지정할 수 있습니다. 위 명령은 사용 방법이며 이번에 성공을 검증한 실행 절차가 아닙니다.

## 관련 기록
- [포트폴리오의 한국어 분석](https://abr4xis.github.io/cases/pwnsec/)
