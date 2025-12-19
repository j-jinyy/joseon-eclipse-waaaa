# @title
import xml.etree.ElementTree as ET
import pandas as pd
import re
from datetime import date, timedelta

# --------------------------------------------------------------------------
# 🌟 마스킹 문자 설정: ' ' (공백) 또는 '_' (언더바) 중 선택하세요.
MASKING_CHAR = '_'
# --------------------------------------------------------------------------

def get_target_dates_id_list(start_date, end_date):
    """
    태조 2년 3월 1일부터 7월 31일까지의 7자리 ID (YYMMDDD, 예: '0204001') 리스트를 생성합니다.
    """
    date_id_list = set()
    current_date = start_date
    while current_date < end_date:
        sillok_date_id = f"02{current_date.month:02d}{current_date.day:03d}"
        date_id_list.add(sillok_date_id)
        current_date += timedelta(days=1)
    return date_id_list

# --------------------------------------------------------------------------
# 2. XML 파일 파싱 및 데이터 추출 함수
# --------------------------------------------------------------------------

def parse_xml_for_sillok_data(file_path, target_date_ids, masking_char):
    extracted_data = []

    try:
        tree = ET.parse(file_path)
        root = tree.getroot()
        print(f"✅ 파일 로드 성공: {file_path}")
    except FileNotFoundError:
        print(f"❌ 오류: 파일을 찾을 수 없습니다. 경로를 확인해주세요: {file_path}")
        return []
    except ET.ParseError:
        print(f"❌ 오류: XML 파싱에 실패했습니다. 파일 구조가 손상되었을 수 있습니다: {file_path}")
        return []

    print("\n--- 날짜 필터링 및 텍스트 마스킹 진행 중 ---")

    for article in root.findall('.//level5'):
        article_id = article.get('id')

        if not article_id or len(article_id) < 12:
            continue

        date_id_seven_digits = article_id[5:12]

        # 날짜 범위 필터링
        if date_id_seven_digits in target_date_ids:

            # 3. 기사 제목 추출
            title_element = article.find('./front/biblioData/title/mainTitle')
            title = title_element.text.strip() if title_element is not None and title_element.text else "제목 없음"

            # 4. 한문 원문 추출 및 마스킹
            paragraph_element = article.find('./text/content/paragraph')
            chinese_text = ""

            if paragraph_element is not None:
                raw_text = ET.tostring(paragraph_element, encoding='unicode')

                # 🌟🌟🌟 수정된 마스킹 로직 🌟🌟🌟

                # <index...> 태그와 그 안의 내용(한자 포함) 전체를 마스킹 문자로 치환합니다.
                # (flags=re.DOTALL: . 문자가 개행문자(\n)도 포함하도록 설정)
                cleaned_text = re.sub(r'<index[^>]*>.*?</index>', masking_char, raw_text, flags=re.DOTALL)

                # <annotation> 태그 및 내용 제거 (이전과 동일)
                cleaned_text = re.sub(r'<annotation[^>]*>.*?</annotation>', '', cleaned_text, flags=re.DOTALL)

                # <paragraph> 태그 자체 제거 및 나머지 정리 (이전과 동일)
                cleaned_text = re.sub(r'<paragraph[^>]*>|.*?:/|</paragraph>', '', cleaned_text)
                cleaned_text = re.sub(r'&lt;.*?&gt;', '', cleaned_text)

                # 여러 개의 공백/언더바가 연속될 경우 하나로 합치고 정리
                chinese_text = re.sub(r'\s+', ' ', cleaned_text).strip()
                chinese_text = re.sub(f'{re.escape(masking_char)}+', masking_char, chinese_text).strip()

            # 5. 결과 저장 (ID에서 날짜 정보를 다시 추출)
            year_part = int(article_id[5:7])
            month_part = int(article_id[7:9])
            day_part = int(article_id[9:12])

            extracted_data.append({
                "ID": article_id,
                "발행일": f"태조 {year_part}년 {month_part}월 {day_part}일",
                "기사제목": title,
                "본문내용_한문_원문_마스킹": chinese_text
            })

    return extracted_data

# --------------------------------------------------------------------------
# 3. 메인 실행 블록
# --------------------------------------------------------------------------

# 1. 대상 기간 설정
START_DATE = date(1393, 3, 1)
EXCLUSIVE_END_DATE = date(1393, 8, 1)

target_dates_list = get_target_dates_id_list(START_DATE, EXCLUSIVE_END_DATE)
XML_FILES_TO_PROCESS = ['2nd_waa_102.xml']

print(f"⭐ 추출 대상 기간: 태조 2년 3월 1일부터 7월 31일 ({len(target_dates_list)}일)")

final_results = []
for file in XML_FILES_TO_PROCESS:
    # 🌟 마스킹 문자 전달 🌟
    results = parse_xml_for_sillok_data(file, set(target_dates_list), MASKING_CHAR)
    final_results.extend(results)

if final_results:
    df_final = pd.DataFrame(final_results)

    df_final.drop_duplicates(subset=['ID'], inplace=True)
    df_final.sort_values(by='ID', inplace=True)

    output_filename = 'a020701_한문원문_최종마스킹_.tsv'
    df_final.to_csv(output_filename, index=False, encoding='utf-8-sig', sep='\t', quoting=1)

    print("\n" + "="*50)
    print(f"🎉 총 {len(df_final)}개의 기사(한문 원문) 추출 및 저장 완료! (마스킹 문자: {MASKING_CHAR})")
    print(f"**결과 파일명: {output_filename}**")
    print("="*50)

    display(df_final.head())

else:
    print("\n수집된 데이터가 없습니다. 파일을 찾지 못했거나 파일 내에 지정된 기간의 기사가 없는지 확인해주세요.")
