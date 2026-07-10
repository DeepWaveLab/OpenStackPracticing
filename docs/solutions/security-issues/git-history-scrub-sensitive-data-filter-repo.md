# Repo 轉公開前的 git 歷史清洗:filter-repo 全歷史抹除敏感資料

**2026-07-10 · git-filter-repo · 適用:任何要轉 public / 對外分享、但歷史含敏感字串或內部檔案的 repo**

## 情境與需求

Repo 要開放(轉 public、開 GitHub Pages、給外部看),但**歷史 commit** 裡有:真實 IP、雲端訂閱 ID、email、示範密碼、不該公開的內部規劃檔。只改最新版(HEAD)沒用——`git log -p`、GitHub 的歷史瀏覽、任何 clone 都翻得到舊內容。需求:**任何歷史版本都追不到**。

## 解法(完整流程)

```bash
# 0. 改寫歷史前必備份(整個 repo 目錄複製一份)
cp -r <repo> /tmp/<repo>-backup-$(date +%H%M%S)

# 1. 要整檔抹除的,先把檔案本體移出 repo 保存(如果還需要它)
mv docs/plans/internal-plan.md ~/safe-place/

# 2. 替換規則檔(舊==>新;每行一條,literal 匹配)
cat > /tmp/replacements.txt << 'EOF'
203.0.113.99==><VM_PUBLIC_IP>
aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee==><subscription-id>
someone@company.com==>you@example.com
RealDemoPassword!==>Sup3rS3cretDemo!
EOF

# 3. 一次改寫全部歷史:抹檔案 + 換字串
git filter-repo --force \
  --invert-paths --path docs/plans/internal-plan.md \
  --replace-text /tmp/replacements.txt

# 4. 驗證(關鍵,見下方清單)後 force push
git remote add origin git@github.com:org/repo.git   # filter-repo 會拔掉 origin!
git push --force -u origin main
```

替換值的慣例:IP 用 **RFC 5737 文件保留網段**(`203.0.113.x`、`198.51.100.x`、`192.0.2.x`——標準的「範例專用 IP」,永遠不會撞到真實主機);識別碼用 `<subscription-id>` 這類佔位符;email 用 `example.com`。

## 驗證清單(每一項都要跑,不能只看 working tree)

```bash
# 每個敏感字串:全歷史逐字搜,必須為 0
git log --all -S"<敏感字串>" --oneline | wc -l

# 被抹除的檔案:任何歷史 commit 都不得出現
git log --all --oneline -- path/to/removed-file.md | wc -l

# 遠端抽驗:直接打 API 看 GitHub 上的實際內容
gh api repos/org/repo/contents/<file> --jq '.content' | base64 -d | grep -c "<敏感字串>"

# 內容沒被替換弄壞(有網站的話)
mkdocs build --strict
```

## 踩雷記錄(實戰四連,每條都真的撞過)

### 1. 完整字串替換抓不到「部分引用」

替換規則寫的是完整 UUID,但文件某處用了**前 8 碼縮寫**當註解(「確認 aaaaaaaa 在列」)——完整字串匹配不到,歷史裡存活。**教訓:驗證時用足夠短的子字串掃**(`git log --all -S"aaaaaaaa"`),掃到再補一輪替換。

### 2. 更早時期的「第二把身分」

清單是憑「現在在用的」記憶列的,結果更早時期的紀錄用的是**另一顆訂閱、另一個 IP、另一套內部命名**——第一輪全漏。**教訓:scrub 清單必須從全 repo 盤點產生**(掃 UUID 形態、公網 IP 形態、email 形態、內部網域字樣的 pattern),不是憑記憶列舉。順帶掃裸露的租戶名/組織縮寫這類「非機密但內部」的字樣。

### 3. filter-repo 會移除 origin remote

這是它的安全設計(防止誤推)。改寫完 `git push` 會失敗,要先 `git remote add origin …` 再 force push。

### 4. force push 後,GitHub 仍可能殘留「不可達物件」

舊 commit 變成 unreachable object,但**知道舊 SHA 的人在 GitHub 執行 GC 前仍可能直接以 SHA 取得**。剛建立、無 fork、無 PR、SHA 沒散佈過的 repo 風險趨近於零;要絕對保證就開 GitHub Support ticket 請他們立即清除。另外:repo 轉移(org A → org B)後舊網址有自動轉址,別以為換了 org 就斷了線索。

### 5. 別把敏感字串寫回 scrub 紀錄本身

清完之後寫文件記錄「清了什麼」時,若把原始字串列出來就全部白做。紀錄一律用佔位符描述(「VM 公網 IP」「訂閱 ID 前 8 碼」),本文即示範。

## 預防(讓下次不用清)

- **文件從第一天就用範例值**:IP 用 RFC 5737 網段、識別碼用 `<placeholder>`、密碼用明顯的假值——教學文件本來就該可照抄而不暴露環境。
- **gitleaks(或同類掃描)進 CI**,在 commit 階段就擋,而不是公開前大掃除。
- 內部規劃文件與對外內容**分 repo 或分目錄並排除發佈**,降低「整包公開」時的清理面。
