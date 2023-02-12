
Folgendes Statement generiert eine Tabelle T1 mit 1 Millionen Rows Testdaten:

```sql
create table t1
as
with v1 as (
select rownum id from dual connect by level <= 10000
)
Select
trunc((rownum-1)/5) order_id,
1+ mod(rownum - 1,5) line_id,
trunc(dbms_random.value(10001,20001)) product_id,
trunc(dbms_random.value(50001,55001)) customer_id,
lpad(rownum,8) v1,
dbms_random.string('p',999) padding
from
v1, v1
where
rownum <= 100000
;
```

Die Daten sehen wie folgt aus:

```
         1          5      11780       50626       10
1l,S *L#3__$ls'(![Fa0?BvEd(4weYj4_X!t&}5P"EZWO%p/5P'-S#WR4hxp'nH?9&bz%>R]k=gnHgTyY<&:\BgsAze-m]p<K>RTCDm"no{|k,Mm%~+c204[!H(s{Dz8\:Dl,C{H-#7HO5bGzqq!n&wMfZH@hh:K+@ixPJ72#C-bEw#{:yt
kJrfDum-u,7t8D!B _zY}5~c5}{J"U>-3\"o"/wn\a-KGDOc&r]zOyX!Hn1FL,;)h9e;Sy1]B)&FZOH;qYERPv.\K37N;3xF{sf{>aBH|cA*J!=;HVy|#Ie5M}jn-*%>|mCx1jcn>g]zxKwti2L>]^\IqRkLlGgyQd$[l!*g>Xw|u^+707.j
iaho@/F}2KhoLbWfM"\)jD<S>ga#iy7V*PirtKZ MsWk<OcD`hA{\K?(JJr**^U=|ScB.2f{u5t't!x~Q# FFW_U{]|~*1=KG[.JbK@W}6$QL8({NsPP66ry1`A9D%G](mailto:.JbK@W%7d6$QL8(%7bNsPP66ry1%60A9D%25G)Q$RXL9WO:%xrohRJQNOAG1EX}Od'PhvP:W+xR_LhxJC'~EDb0qZk
6mwPT8`|#N!U\$lCI~4JxsHFWj[.B@sfWn!ookvU/N](mailto:.B@sfWn!ookvU/N)UN1JHEY-*e$3mX4^vXUcZ>#EMocyXNS H%so}DI9)D(U1$`fIAZ1MRc$?tE"t-.Rkx$k}^x }]:kBd,v2?+'!8%gz:qrRXkt>,B/bYfG t}p7p{R5Q)uE3;g3u^Q8du<i\{&?s?25
*w8)!:0k{hxl(]Q#!az?\diRzvVd;dJ["/N4Vx~qHJ%mSnl^^"^P$|g%oom< [H}xH"^VQtJ-!.0SW1Yx66@b2aq~=QVTGpAs'M*Ab)k#laXUx_i&$]bm*oweT6+yfl!([;t;:F|$wy8,A[&B0:CRa:Z[c6;Ivm+I}Np4&f*|WzzKL})dPNL
H([c?Q1e14([MuH*@GC0?g6>}.9c@4dhr~JnhO`WF;oWs!&;so/HMflKWA@$T7zX1_w6F)afb3h3i<Yr~H,m{R5J@js=g[0Wzhn](mailto:MuH*@GC0?g6%3e%7d.9c@4dhr~JnhO%60WF;oWs!&;so/HMflKWA@$T7zX1_w6F)afb3h3i%3cYr~H,m%7bR5J@js=g%5b0Wzhn):
```

Mittels der "where rownum" Klausel am Ende des Statements kann die Anzahl der Zeilen eingeschränkt werden.

Mittels "dbms_random.string('p',999) padding" kann die Länge der Daten in dieser Spalte (999) definiert werden.

Das Erstellen der Tabelle mit 100.000 Rows dauert ca. 22 Minuten und die Tabelle belegt ca. 120MB.

Tags:

[[HowTo]] - [[SQLPLUS]] - [[Testdaten]]