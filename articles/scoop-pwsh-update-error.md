---
title: "Scoopでpwshが更新できない問題と解決方法（PowerShell 7.6）"
emoji: "🔧"
type: "tech"
topics: ["powershell", "scoop", "windows"]
published: true
---

## 結論

`pwsh` 自身から `scoop update pwsh` を実行すると失敗することがある。
その場合は、**powershell.exe（Windows PowerShell）を別で起動して、そこから実行する** と解決できる。

---

## 発生した問題

PowerShell を更新しようとしたところ、次のエラーが発生した。

```text
scoop update pwsh
pwsh: 7.5.4 -> 7.6.0

ERROR The following instances of "pwsh" are still running. Close them and try again.

 NPM(K)    PM(M)      WS(M)     CPU(s)      Id  SI ProcessName
 ------    -----      -----     ------      --  -- -----------
     71    42.72     105.27       1.31    8956   1 pwsh

Running process detected, skip updating.
```

しかし：

```text
taskkill /F /IM pwsh.exe
エラー: プロセス "pwsh.exe" が見つかりませんでした。
```

という状態だった。

見た目には `pwsh.exe` が見つからないのに、Scoop 側では `pwsh` が実行中と判定されており、更新できなかった。

---

## 原因

これは環境固有の問題というより、
**Scoop が更新処理で `pwsh` を利用する構造に起因する**。

`pwsh` 上で `scoop update pwsh` を実行すると、
更新対象の `pwsh` を使いながら `pwsh` 自身を更新しようとする形になる。

そのため、Scoop が「`pwsh` が使用中」と判断して更新をスキップすることがある。

いわば：

```text
pwsh を使って
pwsh を更新しようとしている
```

という状態。

---

## 解決方法

`pwsh` ではないシェルから実行する必要がある。
今回は `cmd.exe` から `powershell.exe` を起動して更新した。

**1. cmd.exe を起動する**

Win + R → `cmd` → Enter

**2. powershell.exe を起動する**

```cmd
powershell.exe
```

**3. scoop update を実行する**

```powershell
scoop update pwsh
```

これで正常に更新できた。

`cmd.exe` を経由せず、スタートメニューなどから Windows PowerShell を直接起動して実行してもよい。

---

## 確認

```powershell
pwsh -v
```

```text
PowerShell 7.6.0
```

---

## まとめ

- `pwsh` 上で `scoop update pwsh` を実行すると自己更新になって失敗することがある
- 原因は Scoop が更新処理で `pwsh` を利用するため
- `powershell.exe` など `pwsh` 以外のシェルから実行すれば回避できる

---

## 関連Issue

この問題は Scoop の既知の問題として複数 Issue が上がっている。

- [Can't update pwsh: in use by scoop #4838](https://github.com/ScoopInstaller/Scoop/issues/4838)
- [Cannot update `pwsh` because Scoop uses `pwsh` to update #6342](https://github.com/ScoopInstaller/Scoop/issues/6342)
- [Updating pwsh fails, because scoop always uses pwsh during install #6029](https://github.com/ScoopInstaller/Scoop/issues/6029)
- [Scoop fails to update pwsh #3572](https://github.com/ScoopInstaller/Main/issues/3572)
