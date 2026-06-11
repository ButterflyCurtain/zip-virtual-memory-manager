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
  ZIP_Virtual_Memory_Manager_vmdirty_Journal_Spec.txt             … .vmdirty ジャーナル仕様
notes/  レビューメモ・検討メモ
```

## 履歴について

各コミットの日付は、元ファイルの実際の作成・更新日時に合わせてあります。
設計が段階的に育っていった過程を git 履歴として残すためです。
