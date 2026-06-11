# ZIP Virtual Memory Manager — Design Documentation

This repository manages the design documentation for the "ZIP Virtual Memory Manager," which treats ZIP archives as a backing store for virtual memory.
Currently, only the design exists; no implementation code is present yet.

## Structure

```
docs/   Design specifications (with ASCII architecture diagrams)
  ZIP_Virtual_Memory_Manager.txt                                  … Main design document
  ZIP_Virtual_Memory_Manager_Concurrent_Access.txt                … Concurrent access specification
  ZIP_Virtual_Memory_Manager_Diff_Layer_Pressure_Integrity_Detection.txt
                                                                  … Diff Layer / Pressure / Integrity detection specification
  ZIP_Virtual_Memory_Manager_vmdirty_Journal_Spec.txt             … vmdirty journal specification
  ZIP_Virtual_Memory_Manager_vmidx_Index_Spec.txt                 … vmidx seek index specification
notes/  Review notes and discussion memos
```

## Credits

Design review and documentation preparation assisted by AI.
Design decisions are by @ButterflyCurtain.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# ZIP Virtual Memory Manager — 設計ドキュメント

ZIP アーカイブを仮想メモリのバッキングストアとして扱う
「ZIP Virtual Memory Manager」の設計ドキュメントを管理するリポジトリです。
現時点では設計のみで、実装コードはまだありません。

## 構成

```
docs/   設計仕様書 (ASCII アーキテクチャ図含む)
  ZIP_Virtual_Memory_Manager.txt                                  … 本体設計書
  ZIP_Virtual_Memory_Manager_Concurrent_Access.txt                … 並行アクセス仕様
  ZIP_Virtual_Memory_Manager_Diff_Layer_Pressure_Integrity_Detection.txt
                                                                  … Diff Layer / Pressure / 整合性検出仕様
  ZIP_Virtual_Memory_Manager_vmdirty_Journal_Spec.txt             … vmdirty ジャーナル仕様
  ZIP_Virtual_Memory_Manager_vmidx_Index_Spec.txt                 … vmidx シークインデックス仕様
notes/  レビューメモ・検討メモ
```

## 制作について

設計レビューと文書整備に AI支援を利用。
設計判断は @ButterflyCurtain によるもの。