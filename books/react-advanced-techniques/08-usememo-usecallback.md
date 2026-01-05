---
title: "useMemo/useCallback の実践的使い分け"
---

# Chapter 8: useMemo/useCallback の実践的使い分け

## この章で学べること

この章では、useMemo と useCallback の実践的な使い分けを実測データとともに学びます。

- ✅ useMemo の正しい使い方と使うべきケース
- ✅ useCallback の正しい使い方と使うべきケース
- ✅ 依存配列の正しい設定方法
- ✅ 実測データ：70倍〜400倍の高速化事例
- ✅ 使わない方が良いケース
- ✅ よくある失敗：不要なメモ化によるメモリ消費

**前提知識**: useState、useEffect の基本

**所要時間**: 50-60分

---

## 目次

1. [useMemo の基本と使うべきケース](#1-usememo-の基本と使うべきケース)
2. [useCallback の基本と使うべきケース](#2-usecallback-の基本と使うべきケース)
3. [useMemo vs useCallback の使い分け](#3-usememo-vs-usecallback-の使い分け)
4. [依存配列の正しい設定](#4-依存配列の正しい設定)
5. [実測パフォーマンスデータ](#5-実測パフォーマンスデータ)
6. [使わない方が良いケース](#6-使わない方が良いケース)
7. [よくある失敗パターン](#7-よくある失敗パターン)
8. [まとめ](#8-まとめ)

---

<!--
この章は claude-code-skills/react-development/guides/optimization-complete.md の
useMemo/useCallback セクションをベースに執筆します
-->

_この章は執筆中です。次の更新をお待ちください。_
