# Feature Extraction (버전 14)

## 개요
- `feature_extraction_V14.ipynb`에 구현된 파이프라인은 여러 센서 CSV를 읽어 시간/주파수 도메인 특징을 추출합니다.
- InfluxDB CSV 포맷을 가정하며, 멀티 헤더와 필드 누락을 자동 처리하고, 여러 센서를 외부 조인(outer join)으로 동기화합니다.
- 윈도우 단위로 짧은 구간을 처리하므로 센서 상대 상태 비교가 쉬워집니다.

## 구성 요소
1. **설정**
   - `SENSOR_FIELDS`: 처리 대상 센서 필드 리스트로, 이 목록에 포함된 항목만 특징 계산에 참여합니다.
   - `USE_FREQUENCY_DOMAIN`: `True`이면 시간 + 주파수 도메인(총 11개), `False`이면 시간 도메인 5개만 추출합니다.
   - `WINDOW_SIZE`, `WINDOW_OVERLAP`: 특징을 추출할 윈도우 길이와 겹침(초 단위).
   - `OUTPUT_DIR`: 결과 CSV 저장 경로.

2. **CSV 읽기 (`read_influxdb_csv`)**
   - `#group` 헤더가 여러 번 나타나는 CSV에도 대응하며, 각 섹션을 읽어 하나의 DataFrame으로 병합합니다.
   - `_time`, `_field`, `_value`를 기준으로 청소 후 `_field`를 pivot하여 wide format을 생성합니다.
   - `_time`을 시간 순으로 정렬하고, 상대 시간 `Time(s)` 및 샘플링 레이트를 추정해 반환합니다.

3. **특징 추출 (`extract_features`)**
   - 최소 2개 샘플 있을 때 시간 도메인 5가지(표준편차, 피크-투-피크, 크레스트 팩터, 임펄스 팩터, 평균)를 계산합니다.
   - 주파수 도메인을 포함하면 FFT에서 `DominantFreq`, `SpectralCentroid`, `SpectralEnergy`, `SpectralKurtosis`, `SpectralSkewness`, `SpectralStd`도 산출합니다.

4. **다중 센서 처리 (`process_multi_sensor_files`)**
   - 각 센서 CSV를 읽고 `_time`을 인덱스로 하여 리샘플링(`resample_rate` 기본 `100ms`) 후 현재 필드에 따라 외부 조인으로 병합합니다.
   - NaN은 `ffill`/`bfill`으로 보간하며, merge 이후 상대 시간 컬럼(`Time(s)`)을 다시 계산합니다.
   - `window_size`와 `window_overlap` 기준으로 슬라이딩하면서 필드별 특징을 추출하고, `window_id`, `start_time`, `end_time`과 함께 `Field_STD` 등의 열을 추가합니다.

## 실행 예시
```python
sensor_files = {
    'accel_gyro': 'data/raw/.../accel_gyro.csv',
    'pressure': 'data/raw/.../pressure.csv',
}
valid_files = {k: v for k, v in sensor_files.items() if os.path.exists(v)}
features = process_multi_sensor_files(valid_files, resample_rate='78.125ms')
features.to_csv('data/processed/1118_features_misalignment_yellow.csv', index=False)
```
- 실제 존재하는 CSV만 걸러낸 뒤 적절한 리샘플링 주기와 윈도우 설정으로 `process_multi_sensor_files`를 호출하여 결과 CSV를 저장합니다.

## 결과 요약
- 출력 CSV는 각 윈도우별로 `fields_*` 형태의 특징 열을 포함하며, 주파수 도메인을 활성화하면 `*_DominantFreq`~`*_SpectralStd`까지 확장됩니다.
- NaN 보간이 완료되어야 하며, downstream 예측 노트북(`predict_mlp`, `predict_if`)에서 입력으로 사용될 수 있습니다.
