
本專案是一個基於分散式架構的財經資料爬蟲與分析系統。系統透過 `Celery` 與 `RabbitMQ` 進行任務分流，自動爬取證交所的「三大法人買賣超日報（T86）」與「每日收盤行情（MI_INDEX）」，並動態鎖定每日成交量/市值前 10 大與前 20 大的 ETF 進行籌碼力道與價格表現的連動分析。

數據最終將流向 `MySQL` 本地資料庫進行回測分析，並同步上傳至 `Google Cloud BigQuery` 進行雲端大數據分析。

---

## 學習順序建議

如果你是第一次接觸這個專案，建議依序閱讀與執行：

1. `config.py` — 了解環境變數（資料庫帳密、GCP 憑證）怎麼管理。
2. `worker.py` + `tasks.py` — 認識 Celery 核心：解構「爬取 T86」與「計算籌碼集中度」的任務最小範例。
3. `producer.py` — 派送第一個測試任務，親手跑一次任務發送流程。
4. `tasks_crawler_finmind.py` — 觀察真實的財經數據爬蟲與 API 銜接邏輯。
5. `producer_multi_queue.py` — 學習如何將任務分流（如：`twse` 佇列處理 ETF 股價，`tpex` 佇列處理上櫃個股）。
6. `scheduler.py` — 最後把一切串起來，設定每日下午 4:30 自動啟動爬蟲排程。

---

## Docker Compose 檔案說明

專案根目錄包含多個 `.yml` 設定檔，用來一鍵啟動分散式基礎設施與各個服務模組：

### 1. 基礎設施（必須最先啟動）

| 檔案 | 啟動服務 | 說明 |
| --- | --- | --- |
| `rabbitmq.yml` | RabbitMQ + Flower | 本地開發用，使用 `dev` 網路。Flower 為 Celery 任務監控 UI（Port 5555） |
| `rabbitmq-network.yml` | RabbitMQ + Flower | 正式環境版本，使用外部 `my_network`，讓多個獨立的 Compose 互通 |
| `mysql.yml` | MySQL 8.0 + phpMyAdmin | 儲存籌碼與股價資料（Port 3306），phpMyAdmin 提供瀏覽器視覺化管理（Port 8000） |

### 2. Worker（消費者：實際執行爬蟲與指標計算）

| 檔案 | 說明 |
| --- | --- |
| `docker-compose-worker.yml` | 單一 Worker，使用預設 `dev` 網路，適合本地初步測試 |
| `docker-compose-worker-network.yml` | 啟動兩個獨立 Worker（針對 `twse` 與 `tpex` 佇列），使用外部 `my_network` 網路 |
| `docker-compose-worker-network-version.yml` | 支援透過 `DOCKER_IMAGE_VERSION` 環境變數指定 Image 版本，方便進行版本控管與切換 |

### 3. Producer（生產者：負責發送每日爬取清單任務）

| 檔案 | 說明 |
| --- | --- |
| `docker-compose-producer-network.yml` | 執行 `producer_multi_queue.py`，動態計算出目標 ETF 清單並指派任務 |
| `docker-compose-producer-network-version.yml` | 支援環境變數指定 Image 版本的動態生產者環境 |
| `docker-compose-producer-duplicate-network-version.yml` | 執行具備「去重複機制（On Duplicate Key Update）」的進階生產者任務 |

### 4. Scheduler（排程器：自動化大腦）

| 檔案 | 說明 |
| --- | --- |
| `docker-compose-scheduler-network-version.yml` | 啟動 `scheduler.py`，依照台股收盤時間，定時派送每日數據抓取任務 |

### 命名規則小抄
- **`-network`**：使用共用外部網路 `my_network`（需先手動建立），跨 Compose 檔案的容器才能成功連線。
- **沒有 `-network`**：容器獨立使用 Compose 內建的 `dev` 網路，不與外部互通。
- **`-version`**：Image 標籤改為讀取 `${DOCKER_IMAGE_VERSION}` 變數，啟動時需帶入版本號。
- **`-duplicate`**：寫入資料庫時，若遇到相同日期與代號，會自動更新（Upsert）而非報錯。

### 典型架構啟動順序

```bash
# 1. 建立共用網路（僅需執行一次）
docker network create my_network

# 2. 啟動基礎設施（訊息佇列與資料庫）
docker compose -f rabbitmq-network.yml up -d
docker compose -f mysql.yml up -d

# 3. 啟動 Workers（開始監聽任務）
DOCKER_IMAGE_VERSION=0.0.6 docker compose -f docker-compose-worker-network-version.yml up -d

# 4. 啟動排程器（自動定時指派任務）
DOCKER_IMAGE_VERSION=0.0.6 docker compose -f docker-compose-scheduler-network-version.yml up -d

# 5. 瀏覽 http://localhost:5555 透過 Flower 監控爬蟲任務進度
# 6. 瀏覽 http://localhost:8000 透過 phpMyAdmin 查看存入的籌碼數據