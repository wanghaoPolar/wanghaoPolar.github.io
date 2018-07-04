# base64
今天弄了取 ab 实验的一个 task，涉及到 base64 和二进制相关的内容。之前没有接触过，一开始有些抓瞎，这里总结一下。

## base64
选取 base64 作为通用的二进制数据 encode 格式的原因是：
1. 二进制数据不能直接传输，因为一些特殊字符可能会与传输协议本身的字符冲突
2. 如果用 base10 来 encode，那么一个 byte 的二进制数据（0 - 255）需要三个 byte 的 base10 数据来表示。（比如数据 00000001 需要 001 来表示，`001` 每一位都需要一个 byte）
3. 如果用 base16 来 encode，那么一个 byte 的二进制数据需要两个 byte 的 base16 数据来表示
4. 用 base64，每三个 byte 的二进制数据需要四个 byte 的 base64 数据来表示。
5. ascii 只有 95 个字符，64 是最接近 95 的 2 的 power

## atob, btoa
两个命名神秘到让人怀疑是否存在的方法

atob：将 base64 encode 过的字符串 decode 为原始 ascii 字符串
btoa：将 ascii 字符串 encode 为 base64 编码的字符串

## 在浏览器中将 base64 转为 Uint8Array
```typescript
function base64ToUint8Array(base64: string) {
  return new Unit8Array(
    atob(base64) // 转换为原始 ascii 字符串
      .split('')
      .map(char => char.charCodeAt(0)) // 每个 ascii 字符转换为对应的 byte
  )
}
```

## 在 node.js 环境中将 base64 转为 Uint8Array
```typescript
import {Buffer} from 'buffer'

export function base64ToUint8Array(base64: string) {
  return Buffer.from(base64, 'base64')
}
```
