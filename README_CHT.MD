從去年底 Mac OS X 10.5 Leopard 推出開始，在 Mac OS X 上開發輸入法軟體，勢必會面臨這樣一個問題－從 10.4 到 10.5 之間，作業系統所提供的輸入法架構有了重大改變，那麼，應該怎樣以經濟有效的方式，同時兼顧兩個不同的平台與架構？

OpenVanilla 專案的下一個版本 0.9 版（專案代號 Oranje），裡頭採用的 InputMethodKit backporting component（以下簡稱 IMK-Tiger），就是為了解決這個問題而生的實作。

## TSM v.s IMK. Dynamic Loading v.s Client/Server

在 Mac OS X 10.4 以及之前版本的 Mac OS X，所採用的輸入法架構為 [Text
Service Manager](http://developer.apple.com/documentation/Carbon/Reference/Text_Services_Manager/Reference/reference.html)（以下簡稱 TSM），到了 10.5 之後，Apple 加入了 [Input Method Kit](http://developer.apple.com/documentation/Cocoa/Reference/InputMethodKitRefUpdate/Introduction/Introduction.html)（以下簡稱
IMK）這套架構。用最簡單的話描述這兩套架構的差別，就是，TSM 採用的是 Dynamic Loading，而 IMK 則是一套 Client/Server 架構。

![](assets/TSM-VS-IMK.jpg)

### TSM

一套 TSM 輸入法，用 OS X 的術語，是一個 system component，是一個供作業系統載入的 loadable bundle（其實就是一個 dynamic loadable library，像是 Windows 上面的 .dll 檔案，或是 unix 上面用 dlopen() 載入 .so 一樣；不過，OS X 上的習慣是會將個這個 library 或是 excutable binary，與其會用到的各種資源，整理在同一個目錄下，按照固定的慣例整理，這種目錄稱之為 bundle。可參見 [Bundle Programming Guide](http://developer.apple.com/documentation/CoreFoundation/Conceptual/CFBundles/CFBundles.html)）。

每個應用程式在需要使用輸入法的時候，便載入一次這套輸入法的 library，換言之，如果你同時開啟了十套應用程式，而且每個應用程式都需要使用輸入法，那麼，同一套輸入法，也就先後在不同的應用程式中載入了十次。

### IMK

至於一套 IMK 輸入法，本身便是一個應用程式的 Bundle，這個程式與其他應用程式不同的地方在於隱藏了大部分的 UI，例如不顯示在 Dock 上，不顯示主選單（不過，只要你高興，也可以顯示這部份的 UI）。每個應用程式要使用輸入法時，只要啟動一套輸入法應用程式，每個應用程式使用輸入法的方法，則是當應用程式收到了鍵盤與滑鼠事件後，透過跨應用程式通訊，將事件送給輸入法軟體，經過適當的處理之後，再透過跨應用程式通訊，將結果回傳給正在使用輸入法的應用程式。

OS X 應用程式普遍使用 [Distributed Objects](http://developer.apple.com/documentation/Cocoa/Conceptual/DistrObjects/DistrObjects.htm)（分散式物件，以下簡稱 DO）進行跨應用程式通訊。兩套應用程式在使用 DO 通訊時，可以直接互相交換 NSObject 物件，比方說直接把整個 NSArray、NSDictionary 或是 NSImage 丟到對方那邊，寫起來就只像是兩個 Objective-C class 在交換資料而已。以 IMK 輸入法所使用的 DO 來說，則是應用程式中負責文字輸入的那個部分，將所有傳入的事件都以 NSEvent 物件包裝好，送到輸入法那一端，而輸入法先決定要處理那些事件，再回傳應該顯示在輸入緩衝區、或是送入應用程式中的 NSString，或是用一個 NSRange 物件設定輸入緩衝區中應該以較粗的底線強調顯示的區塊的前後範圍。

雖然 10.5 提供了 IMK 架構，但是在 10.5 上面仍然可以繼續使用 TSM 輸入法，而就個人最近所接觸到的 Mac 應用程式專案的經驗，在市面上使用 10.4 的使用者，大概仍然有五成之譜。如果開發 TSM 架構的輸入法就可以顧及 10.4 與 10.5 的使用者，那麼，為什麼要開發 IMK 輸入法？

## Why IMK?

為什麼 Apple 要改變輸入法架構？Apple 一直沒有說清楚，但猜想與接下來 OS X 走向 64 位元有關。

在 64 位元作業系統上，還要可以執行一些既有的 32 位元軟體，如果還要使用 TSM 架構，而 64 位元與 32 位元應用程式會需要不同的 loadable bundle，那麼，同一套輸入法想要在不同軟體中都能夠使用，就得要分別有 32 位元與 64 位元的版本，像目前 Vista 64 上要使用 IME 輸入法，就有這樣的問題（不妨參見[〈關於64位元的嘸蝦米〉](http://ithelp.ithome.com.tw/question/10002274)乙文，而 Vista 64 到目前為止還是讓許多人卻步，原因之一就是許多輸入法還沒有－甚至沒有打算－過渡到
64 位元）。

而如果是換成跨應用程式通訊，便沒有 64/32 位元的問題，64 位元作業系統中的 64 位元應用程式照樣可以使用 32 位元輸入法；而如果在全面改用 64 位元作業系統之前，就開始改變架構，而新架構又是相對簡單，讓既有的第三方輸入法廠商有時間改變設計，那麼也該可以避免 32/64 轉換之間的陣痛。

就開發者與使用者利益的立場，如果現在還在繼續開發 TSM 輸入法，雖然 10.4 與 10.5 上都可以使用，但搞不好下一版作業系統就停止支援－與此相關的是，TSM 輸入法使用 Carbon API，現在誰也說不準下一版作業系統還有沒有 Carbon。此外，即使在 10.5 上可以使用 TSM 輸入法，但是仍然有不少的相容性問題，特別是發生在 10.5 作業系統中使用某些 Carbon 應用程式的時候，常會有無法出字或卡住的狀況。


而單純就開發者的立場，如果您是現在才開始打算寫 OS X 上的輸入法，誠心的建議您直接寫 IMK，寫一套 IMK 輸入法可說是輕鬆愉快，而寫 TSM 輸入法則是受罪。就 API 來說，寫 IMK 輸入法只要自己繼承一個 [IMKInputController](http://developer.apple.com/documentation/Cocoa/Reference/IMKInputController_Class/Reference/IMKInputController_ref.html)class 實作，然後設定好一個 delegate 物件實作[IMKServerInput](http://developer.apple.com/documentation/Cocoa/Reference/IMKServerInput_Additions/Reference/IMKServerInput.html)Protocol 裡頭幾個負責接收事件的 method，在這些 method 中對傳入的 sender物件做些事情，sender就是正在使用輸入法的應用程式，取好一個特定的連線名稱，搞定收工。而如果是TSM，則是又要處理古老又麻煩的 [resourcefork](http://zonble.twbbs.org/archives/2005_07/789.php)，一大堆[命名不知所云狀態又複雜的 Carbon事件](http://zonble.twbbs.org/archives/2005_07/790.php)。

另外，如果你寫的是智慧選字輸入法，如智慧型注音或是拼音，TSM架構又有另外一個問題－因為每份輸入法都是在不同的應用程式中分別載入，所以，假如使用者在這個應用程式中對輸入法做了某些調整，但是在另外一個應用程式所載入的另外一份輸入法，卻不會同時立即更新。將輸入法集中在同一個應用程式還有另一個好處，就是，既然輸入法只使用一份記憶體，那麼便可以開始放心大膽的濫用...呃...善用OS X 所提供的豐富使用者介面元件，比方說，你也可以在輸入法裡頭用個 coverflow 效果...。

而開發 TSM 輸入法最令人髮指的還是如何 debug。在 10.4以及之前的版本中，只要是使用者登入，作業系統就會將各種可以載入的 system component 全部 cache起來，也就是說，只要你改寫了輸入法的任何一小段，想要看到修改過後的行為，在重新編譯後，一定要重新登入一次不可－一個人哪來這麼多青春用來重複登出登入，而前幾年還有人特地為此設置一個讓Window System 快速 crash 掉的快速鍵，以達到快速登出登入的效果。

最可怕的還是輸入法造成載入的應用程式 crash 的狀況－例如忘記把輸入法編成Universal Binary，就拿到其他機器測試－你就可能看到這種讓人歎為觀止的無窮當機的景象：登入，系統叫出 Finder，Finder 載入有問題的輸入法，Finder crashes，螢幕除了桌面底色外什麼都沒有，系統再次叫出 Finder，Finder 再度crash...。當初第一版的 OpenVanilla 就是這樣的，有人在 IRC上面貼出下載網址供親朋好友試用，接著，那天晚上的 IRC頻道突然就變得好安靜。

如果您是現在開始打算寫一套 OS X 輸入法，或是現在打算將其他平台的輸入法port 到 Mac OS X 上，不知道願不願意經歷這種開發過程。

至於IMK，因為輸入法本身是個獨立的應用程式，所以輸入法當掉了，或是其他無法執行的狀況，也就只侷限在這套應用程式而已，不會干擾到其他應用程式，頂多是其他應用程式在使用輸入法時，不管打什麼，都按照鍵盤原本對應的英數字母輸出而已。修改程式碼後要看到結果，也只需要下一行kill 指令，把執行中的輸入法應用程式關閉即可。

但現在又回到最早的問題：10.4 並不支援 IMK，所以現在寫一套 IMK輸入法，就會有市佔率一半的 10.4使用者無法使用，而如果要同時兼顧兩個平台，就必須分別開發 TSM 與 IMK兩個不同架構下的不同版本。而同時開發兩個版本的負擔在於 API實在差距甚大，作業系統向輸入法要的資料型態都不太一樣，比方說，同樣是輸入法選單，TSM跟你要 Carbon menuRef，IMK 跟你要 NSMenu，所以在產生這個選單的時候，就需要分別寫差異甚大的不同程式。已經寫出TSM 版本的輸入法就罷了，但假如是打算寫一套新的輸入法呢？

於是問題很明顯－以 IMK 架構開發輸入法是趨勢，但是 Apple 並沒有在 10.4上提供 IMK。而既然 Apple 不提供 10.4 的 IMK，那麼－

我們就自己做一套 10.4 的 IMK。

## InputMethodKit backporting component

基本的想法非常簡單－10.4 沒有 IMK，但是 IMK 的基礎是一個應用程式 bundle，還有 DO，而 10.4 當然可以跑應用程式 bundle，也有 DO。在 10.5上，作業系統自己做好每個應用程式負責處理文字的部份，與 IMK 之間的 DO連線，在 10.4 上，我們也只要讓每個應用程式負責文字輸入的那個部分與 IMK連線，就可以了。

![](assets/HowIMKTigerWorks.jpg)

所以我們將輸入法分成 client 與 server 兩個部分。輸入法 server 端是一個應用程式 bundle，如果已經在 10.5 上寫好了一個 IMK輸入法，可以幾乎原封不動的變成這個輸入法 server；client 是一個 TSMcomponent，但是這個 TSM component並不處理輸入法的邏輯－比方說那些字根對應到哪個中文，只單純將所有收到的 Carbon 事件翻譯成 NS 事件，透過 DO 丟給輸入法 server 處理，如果 client 發現 server 並不在執行中，就把 server 打開，等 server 啟動後傳入事件，DO 中所使用的 method 完全比照 IMK。server 處理完，便回傳給 TSMclient，client 再把 NS 物件轉換成 Carbon 物件，送入使用者正在使用的應用程式中。

IMK-Tiger 包括定義 DO 連線的一個 header file（因為 10.4 上面沒有 IMKframework，自然也沒有這個 header），以及上述的 TSM 輸入法 client，當中包含各種 Carbon 與 Cocoa 的物件轉換，除事件外，還有像把 IMK的 NSMenu 轉換成 Carbon Menu 等。

如果您已經在 10.5 上，寫好了一套 IMK 輸入法，要透過 InputMethodKitbackporting component，讓您的輸入法變成 10.4/10.5都能使用，其實只要做幾件事情。

* 打開 Xcode，在原本的 project file 中，加入一個 Cocoa loadable bundle 的 target，然後把 IMK-Tiger 的 TSM client 程式碼放進這個 target。這邊還有一些設定要做，暫且不表。

* 把原本的 IMK 輸入法的 target，原封不動複製一份。

* 在上一步新增的 target 中，把你原本 link 的 InputMethodKit.framework 拿掉；然後，在你原本引入 InputMethodKit 的 header 的地方，全都換成 InputMethodKit backporting component 提供的 header。這一步也可以透過 define 的方式來做，例如定義 base SDK 是 10.5 就用 InputMethodKit.framework 的 header，10.4 就用 IMK-Tiger 的 header。

* 再改一下 project file 的設定，加入一個 copy file phrase，再設定一下 dependency，要 Xocde 再編 TSM client 之前先編好 IMK-Tiger server，最後把編出來 IMK-Tiger server ，複製到 TSM client 的 bundle 中的 Shared Support 目錄下。`

做完以上幾步，便可以拿去 10.4 機器上測試看看。

之後，如果要添加任何輸入法的功能，只需要在 server 端下功夫就可以了。

## IMK v.s InputMethodKit backporting component

與 10.5 所提供的 IMK 相較，IMK-Tiger 並沒有完成所有的 IMK實作。InputMethodKit backporting component 完成的是 client side 與 server side 之間的連線，並不包含 IMK 提供給 server 端的各種工具，例如，我們並沒有實作像[IMKCandidates](http://developer.apple.com/documentation/Cocoa/Reference/IMKCandidates_Class/Reference/IMKCandidates_ref.html)所提供的選字窗介面，如果要使用選字窗，就自己在 server 端放一個 NSwindow，自行設計。－而且老實說，就目前可以看到的 OS X輸入法發展，好像所有的第三方輸入法開發者中，也沒有人打算用 OSX本身提供的選字窗就是了...。

另外，就是在 client 在 server 並沒有正常啟動的狀況下，行為略有不同。IMK會變成使用英數字元輸出，IMK-Tiger的行為則是不出字。而就開發上，IMK-Tiger 可能還會更為穩定一些，IMK 是用launchd 啟動，所以當你想要 kill 掉原本的 process，可能 kill 一次還不乾淨，至於 IMK-Tiger 的 server，則是下一個 kill，就 kill 掉了。