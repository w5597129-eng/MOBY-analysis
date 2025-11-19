# Feature Extraction (버전 15)

## 개요
- `feature_extraction_V15.ipynb`는 다중 센서 CSV를 시간/주파수 도메인으로 나눠 윈도우 단위 특징을 추출하고, 예측 파이프라인의 입력으로 정리합니다.
- 이전 버전과 같이 InfluxDB CSV 포맷을 기반으로 다중 헤더, 누락 필드, 리샘플링, 외부 조인을 통해 센서 간 시간 정렬을 보장합니다.
- V15에서는 주파수 통계치에서 NaN이 나오는 상황을 방지하기 위해 기본값을 명시했습니다.

## 주요 단계
1. **환경 설정**
   - `SENSOR_FIELDS`로 추출 범위를 제한하며, `USE_FREQUENCY_DOMAIN` 토글로 5개 또는 11개 특징을 제어합니다.
   - `WINDOW_SIZE`, `WINDOW_OVERLAP`은 윈도우 길이와 겹침이며, `OUTPUT_DIR`은 결과 저장 경로입니다.

2. **CSV 읽기 (`read_influxdb_csv`)**
   - `#group` 헤더 구간이 여러 개일 경우 섹션별로 읽어 모두 병합합니다.
   - `_time`, `_field`, `_value`를 정제하고 `_field`를 컬럼으로 pivot하여 wide format을 생성하며, 정렬과 상대 시간(`Time(s)`)을 계산한 뒤 샘플링 레이트를 추정합니다.

3. **특징 추출 (`extract_features`)**
   - 시간 도메인: 표준편차, 피크-투-피크, 크레스트 팩터, 임펄스 팩터, 평균 5개.
   - 주파수 도메인(사용 시): `DominantFreq`, `SpectralCentroid`, `SpectralEnergy`, `SpectralKurtosis`, `SpectralSkewness`, `SpectralStd`.
   - NaN 방지: 스펙트럼이 작거나 균일한 경우 `SpectralKurtosis=3.0`, `SpectralSkewness=0.0`으로 대체하며, 계산된 값이 NaN이면 같은 기본값으로 대체합니다.

4. **다중 센서 처리 (`process_multi_sensor_files`)**
   - 각 센서 CSV를 `_time` 인덱스로 리샘플링(`resample_rate` 기본 `100ms`)하고 외부 조인으로 병합합니다.
   - NaN은 `ffill`/`bfill`로 보간하고 상대 시간을 재계산합니다.
   - `window_size`와 `window_overlap`만큼 슬라이딩하여 필드별 특징을 추출하고 `window_id`, `start_time`, `end_time` 등을 포함합니다.

## 실행 예시
```python
sensor_files = {
    'accel_gyro': 'data/raw/1118 sensor_data/1118_accel_gyro.csv',
    'pressure': 'data/raw/1118 sensor_data/1118_pressure.csv',
}
valid_files = {k: v for k, v in sensor_files.items() if os.path.exists(v)}
features = process_multi_sensor_files(valid_files, resample_rate='78.125ms')
features.to_csv('data/processed/1118_features_fluctuating.csv', index=False)
```
- 존재하는 센서 파일만 골라 리샘플링 주기와 윈도우 설정에 맞춰 실행하면 `data/processed`에 CSV가 저장됩니다.

## 결과 특징
- 출력 CSV는 각 윈도우별 `field_feature` 열을 가지며, 주파수 도메인을 켜면 `*_DominantFreq`, `*_SpectralCentroid` 등도 포함됩니다.
- `*_SpectralKurtosis`/`*_SpectralSkewness`는 기본값으로 대체되어 downstream 모델이 안정적으로 동작합니다.
- 이후 `predict_mlp.ipynb`/`predict_if.ipynb`에서 사용할 수 있는 입력 형태입니다.
