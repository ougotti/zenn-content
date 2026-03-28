---
title: "Scoopでpwshが更新できない問題と解決方法（PowerShell 7.6）"
emoji: "🔧"
type: "tech"
topics: ["powershell", "scoop", "windows"]
published: false
---

## 結論

`scoop update pwsh` が失敗する場合は
**powershell.exe（Windows PowerShell）から実行する** と解決する。

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

---

## 原因

これは環境の問題ではなく、
**Scoop の構造によるもの**。

Scoop は内部で `pwsh` を使用しているため、
実行中の `pwsh` を自分で更新できない場合がある。

いわば：

```text
pwsh を使って
pwsh を更新しようとしている
```

という状態。

---

## 解決方法

`pwsh` 以外のシェルから実行する必要があるため、`cmd.exe` を起動し、そこから `powershell.exe` を経由して実行する。

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

- `pwsh` 更新失敗はよくある
- 原因は Scoop が `pwsh` を使っているため
- `powershell.exe` から実行すれば解決する

---

## 関連Issue

この問題は Scoop の既知の問題として複数 Issue が上がっている。

- [Can't update pwsh: in use by scoop #4838](https://github.com/ScoopInstaller/Scoop/issues/4838)
- [Cannot update `pwsh` because Scoop uses `pwsh` to update #6342](https://github.com/ScoopInstaller/Scoop/issues/6342)
- [Updating pwsh fails, because scoop always uses pwsh during install #6029](https://github.com/ScoopInstaller/Scoop/issues/6029)
- [Scoop fails to update pwsh #3572](https://github.com/ScoopInstaller/Main/issues/3572)
