# ZIP Virtual Memory Manager — Design Documents

**English** | [日本語](#japanese)

Design documents for a "ZIP Virtual Memory Manager": treating a ZIP archive
as a backing store for virtual memory, with random read/write access into
compressed entries, copy-on-write buffering, and crash-safe commits.

Everything here is design — there is no implementation, and whether one
happens is still undecided (other personal work has my attention right now).
These are old notes I reworked during breaks, and they seemed worth keeping
somewhere proper. If they're useful as a reference, feel free.

## Structure

```
docs/   Design specifications (with ASCII architecture diagrams)
  ZIP_Virtual_Memory_Manager.txt                                  … Main design document
  ZIP_Virtual_Memory_Manager_Concurrent_Access.txt                … Concurrent access specification
  ZIP_Virtual_Memory_Manager_Diff_Layer_Pressure_Integrity_Detection.txt
                                                                  … Diff Layer / Pressure / Integrity detection specification
  ZIP_Virtual_Memory_Manager_vmdirty_Journal_Spec.txt             … vmdirty journal specification
  ZIP_Virtual_Memory_Manager_vmidx_Index_Spec.txt                 … vmidx seek index specification
  IMPLEMENTATION_NOTES.md                                         … a few implementation notes
notes/  Review notes and discussion memos
```

## Credits

Design review and documentation cleanup were AI-assisted.
Design decisions are by @ButterflyCurtain.

---

<a id="japanese"></a>

# ZIP Virtual Memory Manager — 設計ドキュメント

ZIP アーカイブを仮想メモリのバッキングストアとして扱う
「ZIP Virtual Memory Manager」の設計ドキュメントです。
圧縮されたエントリへのランダム読み書き、copy-on-write のバッファリング、
クラッシュセーフな commit あたりを扱っています。

今のところ設計だけで、実装コードはありません。実装するかどうかも未定です。
昔書いたメモを休憩の合間に手直ししたものを、せっかくなので置いておくことにしました。
参考になりそうなら自由にどうぞ。

## 構成

```
docs/   設計仕様書 (ASCII アーキテクチャ図含む)
  ZIP_Virtual_Memory_Manager.txt                                  … 本体設計書
  ZIP_Virtual_Memory_Manager_Concurrent_Access.txt                … 並行アクセス仕様
  ZIP_Virtual_Memory_Manager_Diff_Layer_Pressure_Integrity_Detection.txt
                                                                  … Diff Layer / Pressure / 整合性検出仕様
  ZIP_Virtual_Memory_Manager_vmdirty_Journal_Spec.txt             … vmdirty ジャーナル仕様
  ZIP_Virtual_Memory_Manager_vmidx_Index_Spec.txt                 … vmidx シークインデックス仕様
  IMPLEMENTATION_NOTES.md                                         … ちょっとした実装メモ
notes/  レビューメモ・検討メモ
```

## 制作について

設計レビューと文書整備に AI支援を利用。
設計判断は @ButterflyCurtain によるものです。
