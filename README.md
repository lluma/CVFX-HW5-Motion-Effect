# CVFX HW5 Report  - Team 16

### 1. Take multi-view images by yourselves

Photo 1

![](https://i.imgur.com/1B6KJqc.jpg)

Photo 2

![](https://i.imgur.com/mqYpLSG.jpg)


---

### 2. Show image alignment results between different images

Alignment部分採跟HW4相同的作法，使用ORB descriptor抓出特徵點後將其配對形成Match圖

![](https://i.imgur.com/klyBbDN.jpg)

![](https://i.imgur.com/Wj8spPX.jpg)

---

### 3. Generate the multi-view 3D visual effects - Motion Parallax

我們選擇實作的是Motion Parallax effect，大致方法是利用前面找到的特徵點，去分離移動的前景、後景 & 不變的背景後，將其合成出下一張Frame

* What's motion parallax:

因為視差的關係，導致觀察者在移動時，會覺得距離自己較近的物件，偏移速度會比距離自己較遠的物件快，而我們這次實作的內容為從兩個角度各取一張相片後，將其合成出Parallax effect，同時維持畫面的穩定

* 實作方法：
    需解決的地方在於抓出造成Parallax effect的地方─變動的前後景，然後盡可能維持其他畫面範圍不變，這裡我們參考Google AI Blog關於Motion Stills的解釋，大致步驟為下：
    
    1. 利用Alignment result & cv2.absdiff的協助，圈出重點物件並認知各自移動的方向，目的是提供下一步驟分離前後景時取樣的範圍，以及合成時Inpainting的補齊來源。由圖可以看到重點改變部分即為下方的樹叢 & 後方的樹林稜線，搭配物體偵測便能框出"大致"範圍
    
    ![](https://i.imgur.com/zTSIAbs.jpg)

    
    2. 透過Opencv提供的"GrabCut" algorithm，取上一步驟移動的大致區塊，抓出各自的"前景"，即省略背景不該動/不重要部分，將Parallax effect的物件獨立出來
    
    前景
    
    ![](https://i.imgur.com/cszEWXs.jpg)
    
    後景
    
    ![](https://i.imgur.com/1ynRijW.jpg)
    
    
    3. 最後取Alignment紀錄的相對位置，將物件挪移至Photo2的正確位置，同時在移走的原物件處做內插(Inpainting)補齊

* 分離前後景之演算法 - GrabCut   [[Code Ref](https://docs.opencv.org/3.4.3/d8/d83/tutorial_py_grabcut.html)] [[Algorithm Ref](https://cvg.ethz.ch/teaching/cvl/2012/grabcut-siggraph04.pdf)] 
    
    此方法為基於另一篇Paper提出的GraphCut，進行效果和操作上的修正，不像過去的方法，同時採用了Texture & Edge資訊來做分割，主要的改進部分有三項：
    
    1. GraphCut處理的模型是灰度直方圖，而GrabCut則替換為高斯混合模型（Gaussian Mixture Model, GMM）以直接支援彩色圖片作為Input
    
    2. GraphCut的最佳化分隔過程是一次做到最佳解，而GrabCut改為不斷迭代進行分割和模型參數更新來取得最佳解
    
    ![](https://i.imgur.com/Uia4ua7.png)
    
    
    3. 事前的Mask準備，這也是我們認為GrabCut最大的改進，將原先GraphCut需要先以線條手動標記前景和背景，方能進行分隔，而GrabCut則僅需提供一塊長方區塊將前景框住，就能使用GMM進行建模和分割了，大幅簡化了事前步驟
    
    ![](https://i.imgur.com/iZvGnIA.png)

    
    此外，GMM大致分隔完後，為了取得更自然平滑的邊緣處，加入了border matting的使用，對於同部分的邊緣限制同樣程度的透明度，並直接從前景區域借色、而非混入原背景的方式，以此增加分離後整體的自然感
    
    ![](https://i.imgur.com/0rfhSrK.png)

    至於OpenCV中GrabCut的使用方式很簡單，首先設定一塊長方形的查找範圍，範圍內的物件有可能是前景or背景，但畫面外的必定為後景，這個部分則透過Step 1移動特徵點、取大致長方區塊獲得，將圖片傳入後便會得到結果Mask，Mask與原圖等大小，Pixel value共有四種可能: 
    
    [0: 背景, 1: 前景, 2: 可能為背景, 3: 可能為前景]
    
    接著只要取出1 & 3的地方，便能獨立分離出我們要的前景部分，然後依照相對移動位子做合成，就能達到Motion parallax effect。此外單用正方區塊可能會得到模糊結果(Ex. 草叢中間的湖水部分)，可以額外用Manually marked mask來提供GrabCut更精準的分離結果

* Result:
    * Source: 單純兩張組合成的圖，可以看到因為手震+鏡頭傾斜，Parallax效果不清楚
        
        ![](https://i.imgur.com/Hfd0LYD.gif)
 

    * Output: 可以看到前景(樹叢)和後景(樹林&建築物)一左一右偏移，且前景比後景速度快，中間的湖與湖心亭則維持不動，小缺陷在下方泥土柵欄處合成的不甚完美
    
        ![](https://i.imgur.com/8ithwbk.gif)


### 4. Exploit creativity to add some image processing to enhance effect (Using post-production software is allowed)
    
這裡我們使用了Photoshop的魔術棒及修補工具，想辦法分離下方柵欄處且保留周圍痕跡，然後還原回正確位子上，並透過修補工具消除接縫處的不和諧，使得後至過後的圖片更接近只有目標前景─樹叢在動。

![](https://i.imgur.com/Ta5EEk9.gif)

---

### 5. Bonus - Complete the above 3 different effects
以下盡量都要附一下來源簡介or使用的演算法內容
* Motion parallax - 即上面例子
* Stop motion
    做法相當單純，我們盡量固定相機與物件的距離，以及讓相機面向的方向盡量維持在物件的中心，每隔一段角度後拍下一張圖片，然後將這些圖片串起來
    ![](ow77LCq.gif)
    為了讓串起來的gif可以更平滑流暢，我們還使用了video stabilization工具(https://video-stabilize.com/)，讓物件的抖動程度變小。
    ![](lSyAXtc.gif)
* Live photo

    做法與Parallax雷同，差別在這次沒有前後景，而是需要找出變動的地方(噴泉 & 湖水流動)，再與Ref photo不變背景合成，解決攝影時手震的問題
    gif過大，可能要手動上傳
    * Source: 可以看到畫面有些微晃動
    
    ![](qLvaL2F.gif)
    
    
    * Output: 處理後的結果成功達到穩定背景、只留噴泉及湖水改變的效果
    
    ![](DRIOCto.gif)
    
