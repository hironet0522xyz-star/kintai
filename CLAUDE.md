# 授業・ロボコン勤怠アプリ（kintai.html）

1ファイルのWebアプリ。GitHub Pages（https://hironet0522xyz-star.github.io/kintai/kintai.html）で公開し、利用者はほぼiPhoneから使う。
記録はスマホのブラウザ（localStorage と IndexedDB）にあり、Claude連携中は非公開リポジトリ hironet0522xyz-star/kintai-data の records.json に同期される。
毎晩20:51ごろの予定タスク「勤怠アドバイス（夜）」が records.json の summary を読み、advice.json を書く。

## 改装するときの決まり（守らないと、利用者が改ざんの罰ポイントを受ける）

このアプリには不正対策がある。保存のたびに監査ログ（localStorage の `kintai.audit`）へ、保存データのハッシュを鎖状に記録する。
起動時に保存データが最後の行と一致しないと「改ざん」と判定され、罰ポイント+5になる。

- **保存データをコードで変えるときは、必ず `sysChange("説明", () => { ... })` の中で行う。**
  起動時の移行処理、既定値の変更、データ形式の変更はすべてこれを使う。
  利用者の操作ではないので、猶予も罰も付かず、監査ログには「アプリ」の行として残る。
- **localStorage を直接書かない。** `localStorage.setItem("kintai.v1", …)` を使うと改ざんとして検知される。必ず `save()` を通す。
- **機能を変えたら `APP_VERSION` を上げる。** 起動時に「アプリ更新 旧→新」の行が監査ログに残る。
- **新しい罰の種類を足すときは、`PEN_TYPES` の4番目に導入日（その日の日付）を書く。** 導入日より前の出来事には罰を付けない。
  罰の点数は `penalty.ptsHist` に日付ごとに記録している。計算には `ptsAt(date, type)` を使い、`penalty.pts` は表示用。
- **監査ログの形式を変えるときも、消さずに追記で移行する。** ログの長さが減ると、夜のClaudeチェックが「巻き戻し」と判定する。
- **ルールを甘くする操作は `schedule()` で24時間の猶予に入れる。** 厳しくする操作はすぐ反映してよい。
  甘い・厳しいの判定は `isLooser()` と `tightLoose()` にある。
- **records.json の `summary` と `audit` の形を変えたら、予定タスクのプロンプトも合わせて直す。**
- **課題の証拠写真**：「写真で提出」→ 端末内チェック `analyzeProof()`（暗い・白飛び・無地・ぼやけ・使い回し）→ 通ったものだけ 512px・低画質で `proofs/<課題id>.jpg` に送る。
  夜のタスクが `summary.tasks.proofsPending` の写真だけを見て `advice.json` の `proofs` に可否を書き、見た写真は消す。認められなかった課題はアプリが未提出に戻す。
  トークン節約のため、授業・ロボコンの打刻写真はClaudeに送らない。
- **授業後の課題の自動登録**：`autoTasks()` が、`AUTO_TASK_FROM` 以降に終わった授業ごとに「科目の課題」を締切2日後23:59で登録する（同じ日の同じ科目は1件、休講は除く）。
  その授業の日のうちは、内容・締切・「今回は課題なし」を自由に直せる。翌日以降に締切を延ばすのは `schedule()` を通す。

- **起床チェック**：`wakeInfo(d)` が授業のある日の起きる時刻（最初の授業の `wakeBefore` 分前）を出す。2時間前から +`wakeGrace` 分までに足し算1問で打刻し、間に合わなければ `oversleep`（`WAKE_FROM` から）。`#wake` リンクで打刻タブを開く。
- **二度寝の感知**：iPhoneのショートカット（アラーム停止 → `kintai.html#alarm` を開く）で `recordAlarm()` が `D.alarms` にその朝最初の停止時刻を残す（起きる時刻の4時間前〜最初の授業まで）。
  止めてから `ALARM_GAP`（10分）以内に起床チェックがなければ `snooze`。寝坊（`oversleep`）になった日は二重に付けない。記録したら URL の `#alarm` を消す（次の朝も hashchange が起きるように）。
- **勉強の監視**：勉強タブ。教材を撮って開始 → `studyEvery` 分ごとに成果を撮る → 写真と写真の間（間隔＋`STUDY_LATE`以内）だけを `studyMs()` で数える。
  直前の写真とほぼ同じもの（`tinyDiff` < 1% など）ははじく。3時間報告がなければ `closeStaleStudy()` が自動終了。
  連携中は終了時に最初と最後の写真を並べた1枚を `study/<id>.jpg` に送り、夜のタスクが `advice.json` の `study` に可否を書く。認められない勉強は時間に数えない。
  勉強の写真は最長14日で整理する（`purgePhotos`）。報告間隔を長くするのは猶予（`isLooser` の `studyEvery`）。

## 確認

ブラウザテスト（Playwright、Chromium は /opt/pw-browsers）で、少なくとも次の流れを確かめる。
- 起動
- 打刻（GPSと写真）
- 罰ポイントの計算
- 猶予（24時間後に反映されること）
- 改装後に開き直しても改ざん判定が出ないこと
