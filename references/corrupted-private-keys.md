# 诊断消息平台传输导致的私钥损坏

用户通过 Telegram 等消息平台粘贴 OpenSSH 私钥时，常见损坏模式及诊断方法。

## 常见损坏模式

### 1. 填充空格：`= =` 而非 `==`
Telegram 会在 base64 填充 `=` 之间插入空格。去掉空格即可修复。

### 2. 行边界丢字符
消息平台换行时可能吞掉 1-2 个字符。表现为 base64 总长度 mod 4 ≠ 0。

**快速判断**：拼接所有行后检查 `len(body) % 4`：
- `0` → 长度正常，但内容仍可能有误（见模式 5/6）
- `1` → 丢了 3 个字符或多了 1 个（无法简单修复）
- `2` → 可能丢了 2 个字符
- `3` → 可能丢了 1 个字符

### 3. 公钥/私钥不一致
OpenSSH 私钥格式中，公钥在头部（public section）和私有部分（private section）各出现一次，值必须完全一致。
如果两次出现的公钥不同（如 `7tlUw` vs `7tllJw`），说明内容在传输中被篡改。

**用户指导**：如果用户发了多次，可以交叉比对不同版本，找出哪个字符是对的。例如第一次公钥段写 `7tlUw`，第二次私钥段写 `7tllJw`（多了1字符正好解释行长度差）→ 以更长的那次为准。

### 4. 私钥部分截断
私钥声明长度与实际可用字节数不符。如果解码后总字节数远小于预期（见下方大小参考），密钥不完整。

### 5. 校验和不匹配（check1 ≠ check2）⭐
OpenSSH 私钥格式的 private section 开头有两个 32 位整数 `check1` 和 `check2`，值必须相同。
如果不匹配（如 `a00ec4b8` vs `aa0ec4b8`，只差 1 字节），说明私钥数据在传输中被损坏。

`ssh-keygen -y` 会报 `incorrect passphrase supplied to decrypt private key`（即使密钥无密码 `none`）。
可以强制修复为相同值让 `ssh-keygen -y` 通过，但**这不代表密钥数据本身是正确的**（见模式 6）。

### 6. 格式合法但密码学无效 ⭐⭐
**最重要的诊断**：`ssh-keygen -y` 成功提取公钥（从 public section 读取），并不代表私钥数据正确。
服务器仍可能报 `Permission denied (publickey)`，因为私钥的 ed25519 种子已被损坏，无法生成有效签名。

**必须用 PyNaCl 做种子验证**（见下方诊断脚本）：
- 从 private section 提取 64 字节 seed（前 32 字节是真正的 ed25519 私钥种子）
- 用 PyNaCl 从种子推导公钥
- 对比推导出的公钥和 key file 中声明的公钥是否一致
- 不一致 → **种子已损坏，无法修复，必须重新获取密钥**

## 诊断脚本

### 脚本 1：解析 OpenSSH 二进制格式

当 `ssh-keygen -y -f <key>` 报 `error in libcrypto` 时，用 Python 解析二进制结构定位损坏：

```python
import base64, struct

# 1. 读取密钥文件，提取 base64 body
lines = [l.strip() for l in open('/tmp/keyfile') if not l.startswith('-----')]
body = ''.join(lines).replace(' ', '')
print(f'Total length: {len(body)}, mod 4: {len(body) % 4}')

# 2. 如果 mod 4 != 0，先尝试补 padding
clean = body.rstrip('=')
pad = {0: '', 2: '==', 3: '='}.get(len(clean) % 4, None)
if pad is None:
    print('Cannot fix: mod 4 = 1, truly corrupted')
else:
    decoded = base64.b64decode(clean + pad)

# 3. 解析 OpenSSH 格式
data = bytearray(decoded)

# Magic (15 bytes, no length prefix)
magic = data[0:15]  # b'openssh-key-v1\x00'
off = 15

def read_str(data, off):
    length = struct.unpack('>I', data[off:off+4])[0]
    off += 4
    if off + length > len(data):
        return f'(TRUNCATED: need {length}, have {len(data)-off})', off + length
    return data[off:off+length], off + length

cipher, off = read_str(data, off)    # b'none'
kdf, off = read_str(data, off)       # b'none'
kdfopts, off = read_str(data, off)   # b''
nkeys = struct.unpack('>I', data[off:off+4])[0]  # 1
off += 4

pubblob, off = read_str(data, off)
# 解析公钥
pktype, poff = read_str(pubblob, 0)   # b'ssh-ed25519'
pklen = struct.unpack('>I', pubblob[poff:poff+4])[0]
poff += 4
pub_from_pub_section = pubblob[poff:poff+pklen]
print(f'Public key: {pub_from_pub_section.hex()}')

# 私钥 blob
if off + 4 <= len(data):
    privlen = struct.unpack('>I', data[off:off+4])[0]
    available = len(data) - off - 4
    print(f'Private blob: declared {privlen}, available {available}, missing {privlen - available}')
```

### 脚本 2：校验和 + 种子验证（关键）⭐

当 `ssh-keygen -y` 报 `incorrect passphrase` 或成功但服务器拒绝时，运行此脚本：

```python
import base64, struct

lines = [l.strip() for l in open('/tmp/keyfile') if not l.startswith('-----')]
body = ''.join(lines).replace(' ', '')
decoded = bytearray(base64.b64decode(body))

# 解析到 private section
off = 15  # magic
def read_str(data, off):
    length = struct.unpack('>I', data[off:off+4])[0]
    off += 4
    return data[off:off+length], off + length

_, off = read_str(decoded, off)  # cipher
_, off = read_str(decoded, off)  # kdf
_, off = read_str(decoded, off)  # kdfopts
off += 4  # nkeys
pubblob, off = read_str(decoded, off)  # pubblob
off += 4  # privlen field
privdata = decoded[off:]

# 1. 校验和检查
check1 = privdata[0:4]
check2 = privdata[4:8]
print(f'check1={check1.hex()} check2={check2.hex()} match={check1==check2}')

# 2. 提取公钥（public section）
pktype, poff = read_str(pubblob, 0)
pklen = struct.unpack('>I', pubblob[poff:poff+4])[0]
poff += 4
pub_in_file = bytes(pubblob[poff:poff+pklen])

# 3. 提取私钥 seed（private section）
poff2 = 8  # skip check1+check2
_, poff2 = read_str(privdata, poff2)  # key type
pklen2 = struct.unpack('>I', privdata[poff2:poff2+4])[0]
poff2 += 4
poff2 += pklen2  # skip pub key in private section
seedlen = struct.unpack('>I', privdata[poff2:poff2+4])[0]  # 64
poff2 += 4
full_seed = bytes(privdata[poff2:poff2+seedlen])

# 4. PyNaCl 验证：从种子推导公钥
from nacl.signing import SigningKey
sk = SigningKey(full_seed[:32])  # 前32字节是真正的ed25519种子
derived_pub = sk.verify_key.encode()

print(f'Pub in key file:    {pub_in_file.hex()}')
print(f'Derived from seed:  {derived_pub.hex()}')
print(f'Match: {pub_in_file == derived_pub}')

if pub_in_file != derived_pub:
    print('\n=> 种子已损坏，密钥密码学无效，无法修复')
    print('   ssh-keygen -y 可能成功（从 public section 读公钥）')
    print('   但 SSH 连接会被服务器拒绝（签名验证失败）')
```

**安装 PyNaCl**（如未安装）：`pip install pynacl -q`

## 典型 ed25519 私钥大小参考

- 解码后总字节数取决于 comment 长度，通常 200-420 字节
- 无 comment 或短 comment：约 200-260 字节（~5 行 70 字符 base64）
- 长 comment：可达 400+ 字节
- 公钥部分（pubkey blob）：固定 51 字节
- 私钥部分（privblob）：通常 120-160 字节（含 check1/check2、key type、pub key、64字节 seed、comment、padding）

**判断标准**：
- privblob declared length ≠ actual available bytes → 截断
- check1 ≠ check2 → 内容损坏（可强制修复 checksum 但不代表种子正确）
- checksum 匹配但 PyNaCl 种子验证失败 → 种子损坏（不可修复）

## 损坏诊断决策树

```
ssh-keygen -y -f <key>
├─ 成功提取公钥
│  └─ SSH 连接测试
│     ├─ 成功 → 密钥正常，继续 add + sync-key
│     └─ Permission denied → 运行种子验证脚本
│        ├─ 种子推导公钥 ≠ 文件公钥 → 种子损坏，无法修复
│        └─ 种子推导公钥 == 文件公钥 → 密钥正常，检查远端 authorized_keys
├─ error in libcrypto
│  └─ 运行解析脚本
│     ├─ mod 4 ≠ 0 → 行边界丢字符
│     ├─ privblob 截断 → 密钥不完整
│     └─ 公钥不一致 → 交叉比对多次版本
└─ incorrect passphrase (但 cipher=none)
   └─ check1 ≠ check2 → 内容损坏
      ├─ 强制修复 checksum → ssh-keygen -y 可能通过
      └─ 但仍需种子验证确认
```

## 处理建议

1. 先检查 `= =` 空格问题（最常见、最易修复）
2. 用脚本 1 解析二进制结构，定位损坏区域
3. 如果公钥/私钥不一致 → 交叉比对用户多次发送的版本
4. 如果 check1 ≠ check2 → 可强制修复让 `ssh-keygen -y` 通过，但仍需做种子验证
5. **用脚本 2 做 PyNaCl 种子验证** — 这是判断密钥是否真正可用的最终标准
6. 种子损坏 → **无法修复**，请用户重新导出
7. 建议用户用 `base64 -w0 ~/.ssh/id_ed25519` 输出单行 base64，避免平台换行吞字符
8. 替代方案：如果源机器有密码，直接用 `sshpass` 走密码认证添加，跳过私钥传输
