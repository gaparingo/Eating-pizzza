# Eating-pizzza
피자가게 엑셀파일 자동정렬
# @title 일정산 엑셀 텍스트 변환기 (실행 버튼을 누르세요)
import pandas as pd
import re
import io
from google.colab import files
from datetime import datetime

def process_settlement():
    print("엑셀 파일을 업로드해주세요...")
    uploaded = files.upload()

    for filename in uploaded.keys():
        print(f"\n📄 파일 분석 중: {filename}")
        try:
            # 1. 헤더 위치 찾기 (상위 30줄 스캔)
            # 엑셀을 헤더 없이 일단 읽어옵니다.
            input_io = io.BytesIO(uploaded[filename])
            temp_df = pd.read_excel(input_io, header=None, nrows=30)

            header_row_index = -1

            for i, row in temp_df.iterrows():
                row_str = " ".join(row.astype(str)).replace(" ", "")
                # '주문경로'와 금액 관련 키워드가 있는 줄을 찾습니다.
                if "주문경로" in row_str and any(x in row_str for x in ["주문금액", "결제금액", "매출"]):
                    header_row_index = i
                    break

            if header_row_index == -1:
                # 못 찾으면 첫 번째 줄로 가정
                header_row_index = 0
                print("⚠️ 헤더를 명확히 찾지 못해 첫 번째 줄을 기준으로 처리합니다.")

            # 2. 진짜 헤더로 다시 읽기
            input_io.seek(0) # 파일 포인터 초기화
            df = pd.read_excel(input_io, header=header_row_index)

            # 3. 필요한 컬럼 찾기
            cols = df.columns.astype(str)

            def find_col(keywords):
                for i, col in enumerate(cols):
                    clean_col = col.replace(" ", "")
                    if any(k in clean_col for k in keywords):
                        return col
                return None

            col_path = find_col(["주문경로", "경로", "채널", "앱", "매체", "주문처"])
            col_amount = find_col(["주문금액", "금액", "매출", "결제", "가격", "합계"])
            col_memo = find_col(["메모", "요청", "비고", "사항"])
            col_time = find_col(["주문시간", "시간", "일시", "접수시간"])

            if not col_path or not col_amount:
                print("❌ 필수 컬럼(주문경로, 주문금액)을 찾을 수 없습니다.")
                continue

            # 4. 데이터 가공
            total_amount = 0
            path_counts = {} # { "배민": {"count": 0, "amount": 0}, ... }

            doit_total_count = 0
            other_total_count = 0
            detected_date = None

            for idx, row in df.iterrows():
                # 데이터 정제
                raw_path = str(row[col_path]).strip()

                # 합계 행이나 빈 행 건너뛰기
                if raw_path in ['nan', 'None', ''] or '합계' in raw_path or '총계' in raw_path:
                    continue

                # 금액 처리
                try:
                    raw_amount = str(row[col_amount]).replace(',', '').replace('원', '').strip()
                    amount = int(float(raw_amount)) # float 변환 후 int (소수점 대비)
                except:
                    amount = 0

                # 날짜 추출 (첫 번째 유효한 날짜 사용)
                if detected_date is None and col_time and pd.notna(row[col_time]):
                    time_str = str(row[col_time])
                    # 정규식으로 날짜 패턴 찾기 (YY-MM-DD, YYYY-MM-DD 등)
                    match = re.search(r'(\d{2,4})[-./](\d{1,2})[-./](\d{1,2})', time_str)
                    if match:
                        year, month, day = map(int, match.groups())
                        if year < 100: year += 2000
                        detected_date = f"{year}년 {month}월 {day}일"

                # 메모 확인
                raw_memo = ""
                if col_memo and pd.notna(row[col_memo]):
                    raw_memo = str(row[col_memo])

                # -- 핵심 로직: 먹깨비/두잇 처리 --
                is_mukkebi = "먹깨비" in raw_path
                is_doit = "두잇" in raw_path

                path_name = raw_path
                current_count = 1

                if is_mukkebi:
                    # 메모에서 숫자 추출
                    num_match = re.search(r'(\d+)', raw_memo)
                    if num_match:
                        current_count = int(num_match.group(1))
                    path_name = "두잇" # 먹깨비는 두잇으로 통합
                elif is_doit:
                    path_name = "두잇"

                # 집계
                total_amount += amount

                if path_name == "두잇":
                    doit_total_count += current_count
                else:
                    other_total_count += current_count

                if path_name not in path_counts:
                    path_counts[path_name] = {"count": 0, "amount": 0}

                path_counts[path_name]["count"] += current_count
                path_counts[path_name]["amount"] += amount

            # 날짜 못 찾았을 경우 오늘 날짜
            if not detected_date:
                now = datetime.now()
                detected_date = f"{now.year}년 {now.month}월 {now.day}일"

            # 5. 결과 텍스트 생성
            result_text = f"일정산 {detected_date}\n\n"
            result_text += f"총 매출    : {total_amount:,}원\n\n"

            # 정렬 순서
            priority_order = ["배민", "배달의민족", "배민1", "쿠팡", "쿠팡이츠", "요기요", "요기배달", "땡겨요", "두잇", "포장", "전화", "홀"]

            def sort_key(key):
                for i, p in enumerate(priority_order):
                    if p in key:
                        return i
                return 999 # 우선순위에 없으면 뒤로

            sorted_keys = sorted(path_counts.keys(), key=sort_key)

            for key in sorted_keys:
                data = path_counts[key]
                if key == "두잇":
                    result_text += f"{key} {data['count']}건 {data['amount']:,}원\n"
                else:
                    result_text += f"{key} {data['count']}건\n"

            result_text += f"\n총 {other_total_count}건 + {doit_total_count}건"

            print("-" * 30)
            print("✅ 변환 결과:")
            print("-" * 30)
            print(result_text)
            print("-" * 30)

        except Exception as e:
            print(f"❌ 오류 발생: {e}")

# 함수 실행
if __name__ == "__main__":
    try:
        process_settlement()
    except ImportError:
        # 로컬 환경 등에서 google.colab이 없을 경우 대비
        print("이 코드는 Google Colab 환경에 최적화되어 있습니다.")
