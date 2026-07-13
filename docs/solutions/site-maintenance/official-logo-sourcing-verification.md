---
title: 官方 logo 與吉祥物的安全取得流程
category: site-maintenance
tags: [logo, mascot, github-api, transparency, verification]
symptom: 要幫文件站補專案 logo,抓回來的圖是錯的人、灰底色塊、或在深色模式下隱形
root_cause: 圖片來源與格式沒有驗證紀律——org avatar 不保證是 logo 也不保證透明,repo 內建資產可能是白色版或去品牌版
---

# 官方 logo 與吉祥物的安全取得流程

**適用場景**:幫教學網站/文件站補上各專案的官方 logo 或吉祥物。看起來是五分鐘的小事,實測踩了五顆雷——本文是完整的安全流程。

## 症狀(三次真實翻車)

1. 抓 `avatars.githubusercontent.com/u/2310974` 當 Ceph logo,拿到的是**某個路人的自拍照**。
2. Ceph 的正牌 logo 到手,但帶著灰色底(#f7f7f7),在網頁上是一塊突兀的灰色方塊。
3. Grafana 的 org avatar 是 **JPEG**——根本沒有透明通道;Skyline repo 裡的 `logo-extend.svg` 渲染出來是**白色線條**(深色頂欄用的),淺色底上完全隱形,而且字樣是去品牌的「Cloud」。

## 根因

- GitHub 的 `avatars.githubusercontent.com/u/<數字>` 是**全域使用者 ID 空間**——憑記憶猜組織 ID,拿到誰的頭像都有可能。
- org avatar 只保證「是這個組織的頭像」,**不保證格式**(可能 JPEG)、不保證透明、不保證是正式 logo。
- 專案 repo 裡的圖片資產是**給產品自己用的**:可能是深色介面專用的白色版、可能被改成中性品牌(white-label),不一定適合外部引用。

## 解法:來源優先序(實測有效)

按順序找,找到即停:

1. **專案文件 repo 的 logo 目錄**。OpenStack 系專案的正式標誌藏在
   `<專案>-apiserver 或主 repo/doc/source/images/logo/OpenStack_Project_<Name>-Icon-RGB.png`
   ——Skyline 的九色鹿吉祥物(1576×2519 透明 PNG)就在這裡,官方吉祥物集反而沒有。**吉祥物集找不到 ≠ 沒有,去專案 repo 挖。**
2. **CNCF 專案 → `github.com/cncf/artwork`**。授權明確、icon/horizontal/stacked 齊全、一律透明底。Prometheus 從這裡拿最乾淨。
3. **產品內建 icon**(repo 的 `public/img/*.svg` 之類,如 grafana 的 `grafana_icon.svg`)——可用,但要過下面的雙底色目檢。
4. **org avatar 當最後備援**,而且**必須用 API 正規取得**:
   ```bash
   curl -sL -o logo.png "$(gh api orgs/<org名> --jq .avatar_url)"
   ```
   絕不手拼 `u/<猜的數字>`。

## 上架前的四道檢查(每一道都對應一次翻車)

1. **目檢(硬規則)**:任何圖片入庫前用眼睛看過。PNG/JPG 直接看;**SVG 要先渲染**(headless browser 截圖),且**同時渲染淺色與深色兩種底**——白色版資產只有這樣才會現形。
2. **透明檢查**:`PIL` 讀取,確認 `mode == RGBA` 且角落像素 alpha == 0。JPEG 一律視為不透明。
3. **灰/白底去背**(對純色底的扁平 logo 有效):取 `(0,0)` 的底色,容差 ±12 內的像素全轉透明——Ceph 的灰底就是這樣清掉的:
   ```python
   from PIL import Image
   im = Image.open('logo.png').convert('RGBA')
   bg = im.getpixel((0, 0))[:3]; px = im.load(); tol = 12
   for y in range(im.height):
       for x in range(im.width):
           r, g, b, a = px[x, y]
           if all(abs(c - t) <= tol for c, t in zip((r, g, b), bg)):
               px[x, y] = (r, g, b, 0)
   im.save('logo-transparent.png')
   ```
4. **出處聲明**:頁尾加一行「XX 標誌為 XX 專案之官方資產,此處作社群教學用途」。

## 預防

- 把「圖片必目檢」寫進工作習慣——它擋下的不是美觀問題,是**把陌生人照片發佈到公開網站**這種事故。
- 記住反例:憑記憶的 ID、看起來像官方的 URL、標題正確的檔案,都不算驗證;**只有眼睛看過的圖才算**。
