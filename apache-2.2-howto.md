# apache 2.2.9で動かしてみる。

apacheをarchiveディレクトリからダウンロードする。
とりあえずコンパイル。

    ./configure --prefix=/usr/local/apache2.2.9 --enable-mods-shared=all

これくらい。zlib-develでハマるくらい。

まずはこれで動くんだけど、最低限、httpd-infoを有効化し、lynxを入れて、localhostからのserver-statusへのアクセスを許可しないとapachectlがうまく動かない。まずはここまで。

ScoreBoardFileはhttpd-mpmあたりで有効化する（なぜならこのディレクティブがmpm_commonだから・・・）

次。ftssをゲットするんだが、これがもう公開されていないため、どこかから、なんとかして探しだす。

    http://fossies.org/linux/www/old/ftss-0.9.3.tar.gz

こんなのがあるので、入れてみる。

    ./configure --with-apr-config=/usr/local/apache2.2.9/bin/apr-1-config --with-apxs=/usr/local/apache2.2.9/bin/apxs
    make
    make install
    which ftss

こうなる。

    Usage: ftss /path/to/scoreboard

試しに実行してみる。

    ftss /usr/local/apache2.2.9/logs/apache_status
    16431   _
    16432   _
    16433   _
    16434   _
    16435   _
    17771   _

という訳で、動く。試しに、「telnet localhost 80」とかやると、１つのスロットが「R」になるので、ちゃんとScoreboardファイルが機能していることが分かる。

# apache 2.2.31（最新）で動かない？

2.2系最新で動かなかったというレポートがあるので確認してみる。

apache-2.2.31.tar.gzをダウンロードして、同様にインストール。
2.2.9環境と共存させるため、プレフィックスは分けてみる。

    ./configure --prefix=/usr/local/apache2.2.31 --enable-mods-shared=all

インストールして、2.2.9と同じように、server-status有効化、ScoreBoardFile有効化を行って、動く状態にする。

この状態で、2.2.9でコンパイルされた、ftssを使ってアクセスする。

    ftss /usr/local/apache2.2.31/logs/server_status
    ftss: apr_shm_attach() failed: 2

想定通りだが、shmへのアタッチ失敗という嫌なメッセージ。

ftssをコンパイルしなおす。

    make distclean
    ./configure --with-apr-config=/usr/local/apache2.2.31/bin/apr-1-config --with-apxs=/usr/local/apache2.2.31/bin/apxs
    make
    make install

メッセージ変わらない。使えないのかな。うーん。

フラフラと検索すると、こんなのがあった。

    http://blog.tracpath.com/dev/apache%E3%82%92%E5%85%B1%E6%9C%89%E3%83%A1%E3%83%A2%E3%83%AA%E7%B5%8C%E7%94%B1%E3%81%A7%E3%83%A2%E3%83%8B%E3%82%BF%E3%83%AA%E3%83%B3%E3%82%B0%E3%81%99%E3%82%8B/

ところが、これも、「Failed to attach to a shared memory segment」。

調べた結果：これはshmの作り方が多分2.2.9→2.2.31で変わったためのように見える。

試しに：

     2.2.9 OK
     2.2.27 OK
     2.2.29 NG
     2.2.31 NG

ううん・・・

# つまりAPI不整合と想定。

ここで改めてftssを確認。どうやらOS標準のaprライブラリをリンクしてしまっている。ここのAPI不整合の被疑。

	[root@libra ftss-0.9.3]# ldd ./src/ftss
	        linux-vdso.so.1 =>  (0x00007fff87d5e000)
	        libapr-1.so.0 => /usr/lib64/libapr-1.so.0 (0x00007fa854ed0000)
	        libpthread.so.0 => /lib64/libpthread.so.0 (0x000000341c800000)
	        libc.so.6 => /lib64/libc.so.6 (0x000000341c400000)
	        libuuid.so.1 => /lib64/libuuid.so.1 (0x000000341fc00000)
	        libcrypt.so.1 => /lib64/libcrypt.so.1 (0x0000003421800000)
	        /lib64/ld-linux-x86-64.so.2 (0x000000341c000000)
	        libfreebl3.so => /lib64/libfreebl3.so (0x0000003421400000)
	        libdl.so.2 => /lib64/libdl.so.2 (0x000000341cc00000)

というわけで、LD_LIBRARY_PATHでライブラリ強制変更。

	[root@libra ftss-0.9.3]# export LD_LIBRARY_PATH=/usr/local/apache2.2.31/lib
	[root@libra ftss-0.9.3]# ldd ./src/ftss
	        linux-vdso.so.1 =>  (0x00007ffe299fa000)
	        libapr-1.so.0 => /usr/local/apache2.2.31/lib/libapr-1.so.0 (0x00007f4e173ab000)
	        libpthread.so.0 => /lib64/libpthread.so.0 (0x000000341c800000)
	        libc.so.6 => /lib64/libc.so.6 (0x000000341c400000)
	        librt.so.1 => /lib64/librt.so.1 (0x000000341d000000)
	        libcrypt.so.1 => /lib64/libcrypt.so.1 (0x0000003421800000)
	        /lib64/ld-linux-x86-64.so.2 (0x000000341c000000)
	        libfreebl3.so => /lib64/libfreebl3.so (0x0000003421400000)
	        libdl.so.2 => /lib64/libdl.so.2 (0x000000341cc00000)

これなら動く。

	[root@libra ftss-0.9.3]# ./src/ftss /usr/local/apache2.2.31/logs/apache_status
	29868   _       localhost       localhost       GET /server-status HTTP/1.1
	29869   _
	29870   _       ::1     localhost       GET /
	29871   _       localhost       localhost       GET /server-status HTTP/1.1
	29872   _       localhost       localhost       GET /server-status HTTP/1.1
	29879   _

・・・あれ。ちょっと見た目が違う。でもabで負荷をかけるとちゃんとステータスは変わる模様。

	[root@libra ftss-0.9.3]# ftss /usr/local/apache2.2.31/logs/apache_status
	29868   C       ::1     localhost       GET / HTTP/1.0
	29869   _       ::1     localhost       GET / HTTP/1.0
	29870   _       ::1     localhost       GET / HTTP/1.0
	29871   _       ::1     localhost       GET / HTTP/1.0
	29872   _       ::1     localhost       GET / HTTP/1.0
	29879   W       ::1     localhost       GET / HTTP/1.0

というわけで、単純な環境問題で動かないこともある。

# 結論

ftssをコンパイルするときに、「LD_RUN_PATH」を設定してしまう。出来上がったftssが正しく該当のApacheのバージョンのaprライブラリを参照しているか、lddコマンドで確認する。

本日一番の発見：lynxじゃなく、linksってプログラムがあるという衝撃の事実。
