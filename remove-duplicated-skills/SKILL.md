---
name: remove-duplicated-skills
description: 重複して登録されているskillを表示し、要望に応じて削除するskill
disable-model-invocation: true
---

## 手順

### 1. 重複して登録されているskillのチェック

**1-1. 古いプラグインバージョンのチェック**

```bash
for d in ~/.claude/plugins/cache/claude-plugins-official/*/; do
    name=$(basename "$d")
    count=$(ls -d "$d"*/ 2>/dev/null | wc -l)
    if [ "$count" -gt 1 ]; then
        echo "AFFECTED: $name has $count versions (should be 1)"
        ls -d "$d"*/
    fi
done
```

出力が何もなければ古いバージョンは存在しない。

**1-2. 重複したシンボリックリンクのチェック**

```bash
ls -la ~/.claude/skills/ 2>/dev/null | grep -c "plugins/"
```

0より大きい数が返ればシンボリックリンクが重複している。

### 2. 結果をユーザーに報告する

チェック結果をまとめてユーザーに提示する。重複がない場合はその旨を伝えて終了する。

### 3. 削除の確認と実行

重複が見つかった場合は**必ず**ユーザーに削除してよいか確認してから実行する。

**3-1. 古いバージョンの削除**

```python
import json, os, shutil

with open(os.path.expanduser("~/.claude/plugins/installed_plugins.json")) as f:
    data = json.load(f)

cache = os.path.expanduser("~/.claude/plugins/cache/claude-plugins-official")

for full_name, installs in data["plugins"].items():
    plugin = full_name.split("@")[0]
    active = installs[0]["version"]
    plugin_dir = os.path.join(cache, plugin)
    if os.path.isdir(plugin_dir):
        for ver in os.listdir(plugin_dir):
            path = os.path.join(plugin_dir, ver)
            if os.path.isdir(path) and ver != active:
                print(f"Removing stale: {plugin}/{ver}")
                shutil.rmtree(path)
```

**3-2. 重複したシンボリックリンクの削除**

```bash
find ~/.claude/skills/ -type l -lname "*/plugins/*" -delete 2>/dev/null
```

### 4. 最終結果を表示してClaude Codeの再起動を促す

削除が完了したら結果を報告し、変更を反映させるためにClaude Codeを再起動するよう促すメッセージを表示する。
