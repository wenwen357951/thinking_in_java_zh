# 附錄D 性能


“本附錄由Joe Sharp投稿，並獲得他的同意在這兒轉載。請聯繫`SharpJoe@aol.com`”

Java語言特別強調準確性，但可靠的行為要以性能作為代價。這一特點反映在自動收集垃圾、嚴格的運行期檢查、完整的字節碼檢查以及保守的運行期同步等等方面。對一個解釋型的虛擬機來說，由於目前有大量平臺可供挑選，所以進一步阻礙了性能的發揮。
“先做完它，再逐步完善。幸好需要改進的地方通常不會太多。”（Steve McConnell的《About performance》[16]）
本附錄的宗旨就是指導大家尋找和優化“需要完善的那一部分”。

## D.1 基本方法

只有正確和完整地檢測了程序後，再可著手解決性能方面的問題：

(1) 在現實環境中檢測程序的性能。若符合要求，則目標達到。若不符合，則轉到下一步。

(2) 尋找最致命的性能瓶頸。這也許要求一定的技巧，但所有努力都不會白費。如簡單地猜測瓶頸所在，並試圖進行優化，那麼可能是白花時間。

(3) 運用本附錄介紹的提速技術，然後返回步驟1。

為使努力不至白費，瓶頸的定位是至關重要的一環。Donald Knuth[9]曾改進過一個程序，那個程序把50％的時間都花在約4％的代碼量上。在僅一個工作小時裡，他修改了幾行代碼，使程序的執行速度倍增。此時，若將時間繼續投入到剩餘代碼的修改上，那麼只會得不償失。Knuth在編程界有一句名言：“過早的優化是萬惡之源”（Premature optimization is the root of all evil）。最明智的做法是抑制過早優化的衝動，因為那樣做可能遺漏多種有用的編程技術，造成代碼更難理解和操控，並需更大的精力進行維護。

## D.2 尋找瓶頸

為找出最影響程序性能的瓶頸，可採取下述幾種方法：

### D.2.1 安插自己的測試代碼

插入下述“顯式”計時代碼，對程序進行評測：

```
long start = System.currentTimeMillis();
// 要計時的運算代碼放在這兒
long time = System.currentTimeMillis() - start;
```

利用`System.out.println()`，讓一種不常用到的方法將累積時間打印到控制檯窗口。由於一旦出錯，編譯器會將其忽略，所以可用一個“靜態最終布爾值”（`Static final boolean`）打開或關閉計時，使代碼能放心留在最終發行的程序裡，這樣任何時候都可以拿來應急。儘管還可以選用更復雜的評測手段，但若僅僅為了量度一個特定任務的執行時間，這無疑是最簡便的方法。

`System.currentTimeMillis()`返回的時間以千分之一秒（1毫秒）為單位。然而，有些系統的時間精度低於1毫秒（如Windows PC），所以需要重複`n`次，再將總時間除以`n`，獲得準確的時間。

### D.2.2 JDK性能評測[2]

JDK配套提供了一個內建的評測程序，能跟蹤花在每個例程上的時間，並將評測結果寫入一個文件。不幸的是，JDK評測器並不穩定。它在JDK 1.1.1中能正常工作，但在後續版本中卻非常不穩定。

為運行評測程序，請在調用Java解釋器的未優化版本時加上`-prof`選項。例如：

```
java_g -prof myClass
```

或加上一個程序片（Applet）：

```
java_g -prof sun.applet.AppletViewer applet.html
```

理解評測程序的輸出信息並不容易。事實上，在JDK 1.0中，它居然將方法名稱截短為30字符。所以可能無法區分出某些方法。然而，若您用的平臺確實能支持`-prof`選項，那麼可試試Vladimir Bulatov的“HyperPorf”[3]或者Greg White的“ProfileViewer”來解釋一下結果。

### D.2.3 特殊工具

如果想隨時跟上性能優化工具的潮流，最好的方法就是作一些Web站點的常客。比如由Jonathan Hardwick製作的“Tools for Optimizing Java”（Java優化工具）網站：

http://www.cs.cmu.edu/~jch/java/tools.html

### D.2.4 性能評測的技巧

+   由於評測時要用到系統時鐘，所以當時不要運行其他任何進程或應用程序，以免影響測試結果。

+   如對自己的程序進行了修改，並試圖（至少在開發平臺上）改善它的性能，那麼在修改前後應分別測試一下代碼的執行時間。

+   儘量在完全一致的環境中進行每一次時間測試。

+   如果可能，應設計一個不依賴任何用戶輸入的測試，避免用戶的不同反應導致結果出現誤差。

## D.3 提速方法

現在，關鍵的性能瓶頸應已隔離出來。接下來，可對其應用兩種類型的優化：常規手段以及依賴Java語言。

### D.3.1 常規手段

通常，一個有效的提速方法是用更現實的方式重新定義程序。例如，在《Programming Pearls》（編程珠璣）一書中[14]，Bentley利用了一段小說數據描寫，它可以生成速度非常快、而且非常精簡的拼寫檢查器，從而介紹了Doug McIlroy對英語語言的表述。除此以外，與其他方法相比，更好的算法也許能帶來更大的性能提升——特別是在數據集的尺寸越來越大的時候。欲瞭解這些常規手段的詳情，請參考本附錄末尾的“一般書籍”清單。

### D.3.2 依賴語言的方法

為進行客觀的分析，最好明確掌握各種運算的執行時間。這樣一來，得到的結果可獨立於當前使用的計算機——通過除以花在本地賦值上的時間，最後得到的就是“標準時間”。

| 運算 | 示例 | 標準時間 |
| --- | --- | --- |
| 本地賦值 | `i=n;` | 1.0 |
| 實例賦值 | `this.i=n;` | 1.2 |
| `int`自增 | `i++;` | 1.5 |
| `byte`自增 | `b++;` | 2.0 |
| `short`自增 | `s++;` | 2.0 |
| `float`自增 | `f++;` | 2.0 |
| `double`自增 | `d++;` | 2.0 |
| 空循環 | `while(true) n++;` | 2.0 |
| 三元表達式 | `(x<0) ?-x : x` | 2.2 |
| 算術調用 | `Math.abs(x);` | 2.5 |
| 數組賦值 | a[0] = n; | 2.7 |
| `long`自增 | l++; | 3.5 |
| 方法調用 | `funct();` | 5.9 |
| `throw`或`catch`異常 | `try{ throw e; }`或`catch(e){}` | 320 |
| 同步方法調用 | `synchMehod();` | 570 |
| 新建物件 | `new Object();` | 980 |
| 新建數組 | `new int[10];` | 3100 |


通過自己的系統（如我的Pentium 200 Pro，Netscape 3及JDK 1.1.5），這些相對時間向大家揭示出：新建物件和數組會造成最沉重的開銷，同步會造成比較沉重的開銷，而一次不同步的方法調用會造成適度的開銷。參考資源[5]和[6]為大家總結了測量用程序片的Web地址，可到自己的機器上運行它們。

(1) 常規修改

下面是加快Java程序關鍵部分執行速度的一些常規操作建議（注意對比修改前後的測試結果）。

| 將... | 修改成... | 理由 |
| --- | --- | --- |
| 接口 | 抽象類（只需一個父時） | 接口的多個繼承會妨礙性能的優化 |
| 非本地或數組循環變量 | 本地循環變量  |根據前表的耗時比較，一次實例整數賦值的時間是本地整數賦值時間的1.2倍，但數組賦值的時間是本地整數賦值的2.7倍 |
| 鏈接列表（固定尺寸） | 保存丟棄的鏈接項目，或將列表替換成一個循環數組（大致知道尺寸） | 每新建一個物件，都相當於本地賦值980次。參考“重複利用物件”（下一節）、Van Wyk[12] p.87以及Bentley[15] p.81 |
| `x/2`（或2的任意次冪） | `X>>2`（或2的任意次冪） | 使用更快的硬件指令 |

### D.3.3 特殊情況

+   字符串的開銷：字符串連接運算符`+`看似簡單，但實際需要消耗大量系統資源。編譯器可高效地連接字符串，但變量字符串卻要求可觀的處理器時間。例如，假設`s`和`t`是字符串變量：

```
System.out.println("heading" + s + "trailer" + t);
```

上述語句要求新建一個`StringBuffer`（字符串緩衝），追加參數，然後用`toString()`將結果轉換回一個字符串。因此，無論磁盤空間還是處理器時間，都會受到嚴重消耗。若準備追加多個字符串，則可考慮直接使用一個字符串緩衝——特別是能在一個循環裡重複利用它的時候。通過在每次循環裡禁止新建一個字符串緩衝，可節省980單位的物件創建時間（如前所述）。利用`substring()`以及其他字符串方法，可進一步地改善性能。如果可行，字符數組的速度甚至能夠更快。也要注意由於同步的關係，所以`StringTokenizer`會造成較大的開銷。

+   同步：在JDK解釋器中，調用同步方法通常會比調用不同步方法慢10倍。經JIT編譯器處理後，這一性能上的差距提升到50到100倍（注意前表總結的時間顯示出要慢97倍）。所以要儘可能避免使用同步方法——若不能避免，方法的同步也要比代碼塊的同步稍快一些。

+   重複利用物件：要花很長的時間來新建一個物件（根據前表總結的時間，物件的新建時間是賦值時間的980倍，而新建一個小數組的時間是賦值時間的3100倍）。因此，最明智的做法是保存和更新老物件的字段，而不是創建一個新物件。例如，不要在自己的`paint()`方法中新建一個`Font`物件。相反，應將其聲明成實例物件，再初始化一次。在這以後，可在`paint()`裡需要的時候隨時進行更新。參見Bentley編著的《編程珠璣》，p.81[15]。

+   異常：只有在不正常的情況下，才應放棄異常處理模塊。什麼才叫“不正常”呢？這通常是指程序遇到了問題，而這一般是不願見到的，所以性能不再成為優先考慮的目標。進行優化時，將小的`try-catch`塊合併到一起。由於這些塊將代碼分割成小的、各自獨立的片斷，所以會妨礙編譯器進行優化。另一方面，若過份熱衷於刪除異常處理模塊，也可能造成代碼健壯程度的下降。

+   散列處理：首先，Java 1.0和1.1的標準“散列表”（`Hashtable`）類需要轉換以及特別消耗系統資源的同步處理（570單位的賦值時間）。其次，早期的JDK庫不能自動決定最佳的表格尺寸。最後，散列函數應針對實際使用項（`Key`）的特徵設計。考慮到所有這些原因，我們可特別設計一個散列類，令其與特定的應用程序配合，從而改善常規散列表的性能。注意Java 1.2集合庫的散列映射（`HashMap`）具有更大的靈活性，而且不會自動同步。

+   方法內嵌：只有在方法屬於`final`（最終）、`private`（專用）或`static`（靜態）的情況下，Java編譯器才能內嵌這個方法。而且某些情況下，還要求它絕對不可以有局部變量。若代碼花大量時間調用一個不含上述任何屬性的方法，那麼請考慮為其編寫一個`final`版本。

+   I/O：應儘可能使用緩衝。否則，最終也許就是一次僅輸入／輸出一個字節的惡果。注意JDK 1.0的I/O類採用了大量同步措施，所以若使用象`readFully()`這樣的一個“大批量”調用，然後由自己解釋數據，就可獲得更佳的性能。也要注意Java 1.1的`reader`和`writer`類已針對性能進行了優化。

+   轉換和實例：轉換會耗去2到200個單位的賦值時間。開銷更大的甚至要求上溯繼承（遺傳）結構。其他高代價的操作會損失和恢復更低層結構的能力。

+   圖形：利用剪切技術，減少在`repaint()`中的工作量；倍增緩衝區，提高接收速度；同時利用圖形壓縮技術，縮短下載時間。來自JavaWorld的“Java Applets”以及來自Sun的“Performing Animation”是兩個很好的教程。請記著使用最貼切的命令。例如，為根據一系列點畫一個多邊形，和`drawLine()`相比，`drawPolygon()`的速度要快得多。如必須畫一條單像素粗細的直線，`drawLine(x,y,x,y)`的速度比`fillRect(x,y,1,1)`快。

+   使用API類：儘量使用來自Java API的類，因為它們本身已針對機器的性能進行了優化。這是用Java難於達到的。比如在複製任意長度的一個數組時，`arraryCopy()`比使用循環的速度快得多。

+   替換API類：有些時候，API類提供了比我們希望更多的功能，相應的執行時間也會增加。因此，可定做特別的版本，讓它做更少的事情，但可更快地運行。例如，假定一個應用程序需要一個容器來保存大量數組。為加快執行速度，可將原來的`Vector`（向量）替換成更快的動態物件數組。

(1) 其他建議

+   將重複的常數計算移至關鍵循環之外——比如計算固定長度緩衝區的`buffer.length`。

+   `static final`（靜態最終）常數有助於編譯器優化程序。

+   實現固定長度的循環。

+   使用`javac`的優化選項：`-O`。它通過內嵌`static`，`final`以及`private`方法，從而優化編譯過的代碼。注意類的長度可能會增加（只對JDK 1.1而言——更早的版本也許不能執行字節查證）。新型的“Just-in-time”（JIT）編譯器會動態加速代碼。

+   儘可能地將計數減至0——這使用了一個特殊的JVM字節碼。

## D.4 參考資源

### D.4.1 性能工具

[1] 運行於Pentium Pro 200，Netscape 3.0，JDK 1.1.4的MicroBenchmark（參見下面的參考資源[5]）

[2] Sun的Java文檔頁——JDK Java解釋器主題：

http://java.sun.com/products/JDK/tools/win32/java.html

[3] Vladimir Bulatov的HyperProf

http://www.physics.orst.edu/~bulatov/HyperProf

[4] Greg White的ProfileViewer

http://www.inetmi.com/~gwhi/ProfileViewer/ProfileViewer.html

### D.4.2 Web站點

[5] 對於Java代碼的優化主題，最出色的在線參考資源是Jonathan Hardwick的“Java Optimization”網站：

http://www.cs.cmu.edu/~jch/java/optimization.html

“Java優化工具”主頁：

http://www.cs.cmu.edu/~jch/java/tools.html

以及“Java Microbenchmarks”（有一個45秒鐘的評測過程）：

http://www.cs.cmu.edu/~jch/java/benchmarks.html

### D.4.3 文章

[6] “Make Java fast:Optimize! How to get the greatest performanceout of your code through low-level optimizations in Java”（讓Java更快：優化！如何通過在Java中的低級優化，使代碼發揮最出色的性能）。作者：Doug Bell。網址：

http://www.javaworld.com/javaworld/jw-04-1997/jw-04-optimize.html

（含一個全面的性能評測程序片，有詳盡註釋）

[7] “Java Optimization Resources”（Java優化資源）

http://www.cs.cmu.edu/~jch/java/resources.html

[8] “Optimizing Java for Speed”（優化Java，提高速度）：

http://www.cs.cmu.edu/~jch/java/speed.html

[9] “An Empirical Study of FORTRAN Programs”（FORTRAN程序實戰解析）。作者：Donald Knuth。1971年出版。第1卷，p.105-33，“軟件——實踐和練習”。

[10] “Building High-Performance Applications and Servers in Java:An Experiential Study”。作者:Jimmy Nguyen，Michael Fraenkel，RichardRedpath，Binh Q. Nguyen以及Sandeep K. Singhal。IBM T.J. Watson ResearchCenter,IBM Software Solutions。

http://www.ibm.com/java/education/javahipr.html

### D.4.4 Java專業書籍

[11] 《Advanced Java，Idioms，Pitfalls，Styles, and Programming Tips》。作者：Chris Laffra。Prentice Hall 1997年出版（Java 1.0）。第11章第20小節。

### D.4.5 一般書籍

[12] 《Data Structures and C Programs》（數據結構和C程序）。作者：J.Van Wyk。Addison-Wesly 1998年出版。

[13] 《Writing Efficient Programs》（編寫有效的程序）。作者：Jon Bentley。Prentice Hall 1982年出版。特別參考p.110和p.145-151。

[14] 《More Programming Pearls》（編程珠璣第二版）。作者：JonBentley。“Association for Computing Machinery”，1998年2月。

[15] 《Programming Pearls》（編程珠璣）。作者：Jone Bentley。Addison-Wesley 1989年出版。第2部分強調了常規的性能改善問題。 [16] 《Code Complete:A Practical Handbook of Software Construction》（完整代碼索引：實用軟件開發手冊）。作者：Steve McConnell。Microsoft出版社1993年出版，第9章。

[17] 《Object-Oriented System Development》（物件導向系統的開發）。作者：Champeaux，Lea和Faure。第25章。

[18] 《The Art of Programming》（編程藝術）。作者：Donald Knuth。第1卷“基本算法第3版”；第3卷“排序和搜索第2版”。Addison-Wesley出版。這是有關程序算法的一本百科全書。

[19] 《Algorithms in C:Fundammentals,Data Structures, Sorting,Searching》（C算法：基礎、數據結構、排序、搜索）第3版。作者：RobertSedgewick。Addison-Wesley 1997年出版。作者是Knuth的學生。這是專門討論幾種語言的七個版本之一。對算法進行了深入淺出的解釋。
