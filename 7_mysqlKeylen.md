# EXPLAIN 実行計画の key_len を解釈する
> EXPLAIN の key_len 列はどのように解釈されますか?

EXPLAIN 実行プランには、このクエリで選択されたインデックスの長さをバイト単位で示すために使用される列 key_len があります。通常、これを使用して、結合インデックスの列がいくつ選択されているかを判断できます。

ここで、key_len サイズの計算規則は次のとおりです。

* 通常、key_len はインデックス列の型のバイト長と同じです。たとえば、int 型は 4 バイト、bigint は 8 バイトです。
* 文字列型の場合は、文字セット要素も考慮する必要があります。たとえば、CHAR(30) UTF8 の場合、key_len は少なくとも 90 バイトです。
* 列タイプが定義されているときに NULL が許可されている場合、その key_len に 1 バイトを追加する必要があります。
* 列の型が VARCHAR などの可変長型の場合 (TEXT\BLOB では、列全体でインデックスを作成することはできません。部分的なインデックスが作成された場合は、動的な列の型と見なされます)、その key_len が必要です。追加される 2 バイト。 

|  列タイプ   | key_len  | 述べる | 
|---| ----  |-----|
| id int  | key_len = 4+1 = 5 |NULL許可します，加1-byte|
| id int not null  | key_len = 4 |NULL 許可されていません|
| user char(30) utf8  | key_len = 30*3+1 |NULL許可します|
| user varchar(30) not null utf8  | key_len = 30*3+2 |动态列类型，加2-bytes|
| user varchar(30) utf8  | key_len = 30*3+2+1 |動的列タイプ、2 byteを追加、NULL を許可、1 byteを追加|
| detail text(10) utf8  | key_len = 30*3+2+1 |TEXT 列のインターセプトされた部分は、動的な列の型に 2byteを加えたものと見なされ、NULL が許可されます。|

* 列タイプ,    key_len  , 述べる 
* id int  , key_len = 4+1 = 5, NULL許可します，加1-byte
* id int not null  , key_len = 4 ,NULL 許可されていません


| user char(30) utf8  | key_len = 30*3+1 |NULL許可します|
| user varchar(30) not null utf8  | key_len = 30*3+2 |动态列类型，加2-bytes|
| user varchar(30) utf8  | key_len = 30*3+2+1 |動的列タイプ、2 byteを追加、NULL を許可、1 byteを追加|
| detail text(10) utf8  | key_len = 30*3+2+1 |TEXT 列のインターセプトされた部分は、動的な列の型に 2byteを加えたものと見なされ、NULL が許可されます。|


**key_len は、WHERE の条件付きフィルターで選択されたインデックス列のみを示し、ORDER BY/GROUP BY で選択されたインデックス列は含まれないことに注意してください。
たとえば、結合インデックス idx1(c1, c2, c3) があり、3 つの列すべてが INT NOT NULL の場合、次の SQL 実行計画では、key_len の値は 12 ではなく 8 になります。：**

> SELECT…WHERE c1=? AND c2=? ORDER BY c1;