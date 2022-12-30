---
tags: [python]
title: 利用Python QRCode模块快速生成二维码
created: '2022-12-30T11:04:32.170Z'
modified: '2022-12-30T11:12:05.672Z'
---

利用Python QRCode模块快速生成二维码

安装qrcode相关模块：
```bash
pip install qrcode
pip install Image
```

生成二维码的Python实现：
```python
import qrcode

def qrCodeGenerator(data, qrcode_file):
  qr = qrcode.QRCode(
    version=1,
    error_correction=qrcode.constants.ERROR_CORRECT_H,
    box_size=12,
    border=4
  )

  qr.add_data(data)
  qr.make(fit=True)

  img = qr.make_image()
  img.save(qrcode_file)
  img.show()


def main():
  data = str(input('请输入文本（例如https://www.baidu.com）：'))
  qrcode_file = r'D:\Images\qrcode.png'
  qrCodeGenerator(data, qrcode_file)
  print('二维码已保存到指定路径下：', qrcode_file)

if __name__ == "__main__":
  main()
```
