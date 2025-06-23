# Medusa `backup` 工作流程

本文件說明執行 `medusa backup` 時程式內部的大致流程，協助了解備份命令完成備份工作的步驟。

1. **啟動指令**
   - CLI 透過 [`backup`](../medusa/medusacli.py#L120-L141) 呼叫 [`handle_backup`](../medusa/backup_node.py#L78-L137) 並解析參數。
   - 未指定 `--backup-name` 時會以當前時間產生名稱。
   - `handle_backup` 會建立一個臨時檔案標記，以避免同一節點同時執行多個備份。

2. **初始化與前置作業**
   - 在 [`start_backup`](../medusa/backup_node.py#L140-L199) 中依設定檔建立 `Storage` 與 `Cassandra` 物件。
   - 取得 schema、token map 及 server 版本後寫入 `NodeBackup`。
   - 呼叫 [`add_backup_start_to_index`](../medusa/index.py#L96-L103) 於儲存端記錄備份開始；若啟用 `--in-stagger` 會等待前一節點完成。

3. **建立快照**
   - [`do_backup`](../medusa/backup_node.py#L221-L255) 預設呼叫 `cassandra.create_snapshot()` 產生新快照；若使用 `--use-existing-snapshot` 則透過 `get_snapshot()` 取得現有快照。
   - 進入 snapshot context 後會在離開時自動清理（除非使用 `--keep-snapshot`）。

4. **檔案判斷與上傳**
   - [`backup_snapshots`](../medusa/backup_node.py#L301-L358) 會對每個 keyspace/table 的 snapshot 目錄列出檔案，並比對外部儲存是否已存在。
   - 在 *差異備份* 模式下，已上傳且內容相同的 SSTable 只會寫入 manifest，不會重新上傳；若檔案不存在或雜湊不同則上傳。
   - 在 *完整備份* 模式下則會將所有檔案複製到儲存空間。
   - 需要重複上傳或已存在檔案的判斷邏輯位於 [`check_already_uploaded`](../medusa/backup_node.py#L365-L405)。
   - 每個表的備份結果（檔案路徑、MD5 及大小）會加入 manifest。

5. **完成備份**
   - 若為 DSE，`do_backup` 會另外呼叫 `create_dse_snapshot()` 處理 DSE snapshot 的備份。
   - 備份完成後，[`add_backup_finish_to_index`](../medusa/index.py#L104-L110) 與 [`set_latest_backup_in_index`](../medusa/index.py#L113-L118) 更新 index。
   - 透過 [`print_backup_stats`](../medusa/backup_node.py#L258-L284) 與 [`update_monitoring`](../medusa/backup_node.py#L286-L299) 輸出統計與監控指標。

6. **收尾**
   - 快照 context 結束後會依設定自動刪除快照。
   - `handle_backup` 的 `finally` 區塊中移除「備份進行中」標記檔案，代表此次備份流程已結束。

以上即為 `medusa backup` 在單一節點執行時的主要流程。叢集備份 (`medusa backup-cluster`) 則會在所有節點先以平行方式建立快照，再依序上傳各節點的資料。
