<template>
  <div class="mapDiv">
    <canvas
      class="map"
      id="gobangMap"
      @click="onMouseClick"
    > </canvas>
    <div class="button-group">
      <div
        class="button"
        @click="repent()"
      >悔棋</div>
      <div
        class="button"
        @click="reset()"
      >
        <font v-if="playing">重新开始</font>
        <font v-else>开始游戏</font>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";

@Component
export default class Map extends Vue {
  canvasContext!: CanvasRenderingContext2D; //画布对象
  canvasWidth: number = 200; //画布宽度
  canvasHeight: number = 200; //画布高度
  latticeWidth: number = 200; //小格子宽度
  latticeHeight: number = 200; //小格子高度
  gameMode: number = 0; //0-人人对战 1-人机对战
  whoDropping: boolean = true; //true黑子 false白子
  playing: boolean = true; //游戏进行状态
  stepStack: Array<[number, number, boolean]> = []; //记步栈
  matrix: Array<Array<number>> = new Array<Array<number>>(); //棋盘矩阵0空1黑2白
  gameAIInvID: number = 0;
  @Prop({ default: 15 }) private size!: number;
  /**
   * mounted
   * @description:
   *   vue生命周期挂载完成
   * @return: {void}
   */
  mounted() {
    //获取画布对象
    let c: HTMLCanvasElement = document.getElementById(
      "gobangMap"
    ) as HTMLCanvasElement;
    //获取画布矩形对象
    let c_rect: ClientRect = c.getBoundingClientRect() as ClientRect;
    //获取画布2D对象
    this.canvasContext = c.getContext("2d") as CanvasRenderingContext2D;
    //设置画布内容宽高为实际宽高
    this.canvasWidth = this.canvasContext!.canvas.width = Math.round(
      c_rect.width
    );
    this.canvasHeight = this.canvasContext!.canvas.height = Math.round(
      c_rect.height
    );
    //计算每个小格子的边长
    this.latticeWidth = this.canvasWidth / (this.size + 1);
    this.latticeHeight = this.canvasHeight / (this.size + 1);
    //构建二维矩阵
    for (let i: number = 0; i < this.size; i++) {
      //构建矩阵行
      this.matrix.push([]);
      //构建矩阵列
      for (let j = 0; j < this.size; j++) {
        this.matrix[i].push(0);
      }
    }
    //窗口大小发生变化时重新计算尺寸
    window.onresize = () => {
      //获取画布矩形对象
      let c_rect: ClientRect = c.getBoundingClientRect() as ClientRect;
      //设置画布内容宽高为实际宽高
      this.canvasWidth = this.canvasContext!.canvas.width = Math.round(
        c_rect.width
      );
      this.canvasHeight = this.canvasContext!.canvas.height = Math.round(
        c_rect.height
      );
      //计算每个小格子的边长
      this.latticeWidth = this.canvasWidth / (this.size + 1);
      this.latticeHeight = this.canvasHeight / (this.size + 1);
      //重新绘制棋盘
      this.drawChessboard();
    };
    //重新绘制棋盘
    this.drawChessboard();
    //接收当前模式号
    this.gameMode = parseInt(this.$route.params.mode) | 0;
    //启动游戏AI
    if (this.gameMode !== 0) {
      this.gameAIInvID = setInterval(() => {
        this.gameAI();
      }, 500);
    }
  }
  /**
   * destroyed
   * @description:
   *   vue生命周期销毁完成
   * @return: {void}
   */
  destroyed() {
    this.playing = false;
    clearInterval(this.gameAIInvID);
    this.gameAIInvID = 0;
  }
  /**
   * onMouseClick
   * @description:
   *   用户点击棋盘操作
   * @param: {MouseEvent} 鼠标事件 鼠标点击相关参数
   * @return: {void}
   */
  onMouseClick(event: MouseEvent): void {
    let x: number = Math.round(
      ((event.offsetX + 10) * 1.0) / this.latticeWidth
    );
    let y: number = Math.round(
      ((event.offsetY + 10) * 1.0) / this.latticeHeight
    );
    //超出棋盘范围了
    if (x < 1 || x > this.size || y < 1 || y > this.size) return;
    // 人机人白没轮到自己，return
    if (this.gameMode === 1 && this.whoDropping == true) {
      return;
    }
    // 人机人黑没轮到自己，return
    if (this.gameMode === 2 && this.whoDropping === false) {
      return;
    }
    if (!this.playing) {
      alert("请点击开始游戏按钮！");
      return;
    }
    this.dropChessman(x - 1, y - 1);
  }
  /**
   * calcPointScore
   * @description:
   *   计算当前方向分值
   * @param: {number} count 两侧相同点个数
   * @param: {boolean} who true表示己方棋子，false表示对方棋子
   * @return: {number} 分数 返回当前方向分值
   */
  calcPointScore(count:number,who:boolean):number{
    switch (count) {
      case 0:
        return 0;
      case 1:
        return who?800:1000;
      case 2:
        return who?5000:3000;
      case 3:
        return who?50000:10000;
      default://五连珠
        return who?1000000:100000;
    }
  }
  /**
   * calcPlacementCoordinate
   * @description:
   *   计算落子点坐标
   * @param: {number} mineNumber 1黑 2白
   * @param: {number} enemyNumber 1黑 2白
   * @return: {object} {x:横坐标,y:纵坐标}
   */
  calcPlacementCoordinate(mineNumber: number,enemyNumber: number): { x: number; y: number } {
    let x!: number, y!: number;
    let maxMine = {
      x: -1,
      y: -1,
      score: -1
    };
    let maxEnemy = {
      x: -1,
      y: -1,
      score: -1
    };
    //遍历每一个空节点
    for (let i = 0; i < this.size; i++) {
      for (let j = 0; j < this.size; j++) {
        //边缘1圈
        /* if(i === 0 || j === 0 || i === this.size - 1 || j === this.size - 1){
          
        } */
        if (this.matrix[i][j] !== 0) {
          continue;
        }
        //left-top
        let count_lt = 0;
        let count_t = 0;
        let count_rt = 0;
        let count_l = 0;
        let count_r = 0;
        let count_lb = 0;
        let count_b = 0;
        let count_rb = 0;
        let score_sum = 0;
        //计算当前空点8个方向上的己方连子个数
        for (
          let _x = i - 1, _y = j - 1;
          _x >= 0 && _y >= 0 && this.matrix[_x][_y] === mineNumber;
          _x--, _y--
        ) {
          count_lt++;
        }
        for (
          let _x = i, _y = j - 1;
          _y >= 0 && this.matrix[_x][_y] === mineNumber;
          _y--
        ) {
          count_t++;
        }
        for (
          let _x = i + 1, _y = j - 1;
          _x < this.size && _y >= 0 && this.matrix[_x][_y] === mineNumber;
          _x++, _y--
        ) {
          count_rt++;
        }
        for (
          let _x = i - 1, _y = j;
          _x >= 0 && this.matrix[_x][_y] === mineNumber;
          _x--
        ) {
          count_l++;
        }
        for (
          let _x = i + 1, _y = j;
          _x < this.size && this.matrix[_x][_y] === mineNumber;
          _x++
        ) {
          count_r++;
        }
        for (
          let _x = i - 1, _y = j + 1;
          _x >= 0 && _y < this.size && this.matrix[_x][_y] === mineNumber;
          _x--, _y++
        ) {
          count_lb++;
        }
        for (
          let _x = i, _y = j + 1;
          _y < this.size && this.matrix[_x][_y] === mineNumber;
          _y++
        ) {
          count_b++;
        }
        for (
          let _x = i + 1, _y = j + 1;
          _x < this.size && _y < this.size && this.matrix[_x][_y] === mineNumber;
          _x++, _y++
        ) {
          count_rb++;
        }
        //计算当前空位己方得分
        score_sum = this.calcPointScore(count_lt + count_rb, true) + 
        this.calcPointScore(count_t + count_b, true) + 
        this.calcPointScore(count_l + count_r, true) + 
        this.calcPointScore(count_lb + count_rt, true);
        //保存己方得分最大点的坐标和分值
        if(score_sum > maxMine.score){
          maxMine.score = score_sum;
          maxMine.x = i;
          maxMine.y = j;
        }
        count_lt = 0;
        count_t = 0;
        count_rt = 0;
        count_l = 0;
        count_r = 0;
        count_lb = 0;
        count_b = 0;
        count_rb = 0;
        //计算当前空点8个方向上的敌方连子个数
        for (
          let _x = i - 1, _y = j - 1;
          _x >= 0 && _y >= 0 && this.matrix[_x][_y] === enemyNumber;
          _x--, _y--
        ) {
          count_lt++;
        }
        for (
          let _x = i, _y = j - 1;
          _y >= 0 && this.matrix[_x][_y] === enemyNumber;
          _y--
        ) {
          count_t++;
        }
        for (
          let _x = i + 1, _y = j - 1;
          _x < this.size && _y >= 0 && this.matrix[_x][_y] === enemyNumber;
          _x++, _y--
        ) {
          count_rt++;
        }
        for (
          let _x = i - 1, _y = j;
          _x >= 0 && this.matrix[_x][_y] === enemyNumber;
          _x--
        ) {
          count_l++;
        }
        for (
          let _x = i + 1, _y = j;
          _x < this.size && this.matrix[_x][_y] === enemyNumber;
          _x++
        ) {
          count_r++;
        }
        for (
          let _x = i - 1, _y = j + 1;
          _x >= 0 && _y < this.size && this.matrix[_x][_y] === enemyNumber;
          _x--, _y++
        ) {
          count_lb++;
        }
        for (
          let _x = i, _y = j + 1;
          _y < this.size && this.matrix[_x][_y] === enemyNumber;
          _y++
        ) {
          count_b++;
        }
        for (
          let _x = i + 1, _y = j + 1;
          _x < this.size && _y < this.size && this.matrix[_x][_y] === enemyNumber;
          _x++, _y++
        ) {
          count_rb++;
        }
        //计算当前空位敌方得分
        score_sum = this.calcPointScore(count_lt + count_rb, false) + 
        this.calcPointScore(count_t + count_b, false) + 
        this.calcPointScore(count_l + count_r, false) + 
        this.calcPointScore(count_lb + count_rt, false);
        //保存敌方得分最大点的坐标和分值
        if(score_sum > maxEnemy.score){
          maxEnemy.score = score_sum;
          maxEnemy.x = i;
          maxEnemy.y = j;
        }
      }
    }
    //返回得分最大点的坐标
    if(maxEnemy.score > maxMine.score){
      x = maxEnemy.x;
      y = maxEnemy.y;
    }else{
      x = maxMine.x;
      y = maxMine.y;
    }
    //如果最大得分为0，就落子在中间。
    if(maxEnemy.score === 0){
      x = Math.round(this.size/2);
      y = Math.round(this.size/2);
    }
    return {
      x: x,
      y: y
    };
  }
  /**
   * gameAI
   * @description:
   *   游戏AI，计算落子
   * @return: {void}
   */
  gameAI(): void {
    //如果游戏没开始就return
    if (!this.playing) {
      return;
    }
    //人机机先模式且当前轮到黑子
    if (this.gameMode === 1 && this.whoDropping == true) {
      let pos = this.calcPlacementCoordinate(1,2);
      this.dropChessman(pos.x, pos.y);
    }
    //人机人先且当前轮到白子
    if (this.gameMode === 2 && this.whoDropping === false) {
      let pos = this.calcPlacementCoordinate(2,1);
      this.dropChessman(pos.x, pos.y);
    }
  }
  /**
   * repent
   * @description:
   *   悔棋
   * @return: {void}
   */
  repent(): void {
    if(this.gameMode === 0 || this.playing === false){
      //栈空了，return
      if (this.stepStack.length < 1) return;
      //最近一步棋出栈
      let [x, y, who] = this.stepStack.pop() as [number, number, boolean];
      //清除矩阵对应位置的记录
      this.matrix[x][y] = 0;
      //切换棋手
      this.whoDropping = who;
    }else if(this.gameMode === 1 && this.whoDropping === false){
      //人机机先模式，人执棋可以悔棋
      //最近2步棋出栈
      let [x1, y1, who1] = this.stepStack.pop() as [number, number, boolean];
      let [x2, y2, who2] = this.stepStack.pop() as [number, number, boolean];
      //清除矩阵对应位置的记录
      this.matrix[x1][y1] = 0;
      this.matrix[x2][y2] = 0;
    }
    //重新绘制棋盘
    this.drawChessboard();
  }
  /**
   * reset
   * @description:
   *   清空落子记录栈，重新绘制棋盘
   * @return: {void}
   */
  reset(): void {
    //清空二维矩阵
    for (let i in this.matrix) {
      for (let j in this.matrix[i]) {
        this.matrix[i][j] = 0;
      }
    }
    //清空落子记录栈
    this.stepStack.length = 0;
    //重新绘制棋盘
    this.drawChessboard();
    //落子权回到黑子
    this.whoDropping = true;
    //状态切换为游戏中
    this.playing = true;
  }
  /**
   * drawChessboard
   * @description:
   *   绘制棋盘和棋子
   * @return: {void}
   */
  drawChessboard(): void {
    //设置背景填充色
    this.canvasContext!.fillStyle = "RGB(194,149,107)"; //"RGB(214,159,64)";
    //绘制矩形背景
    this.canvasContext!.fillRect(0, 0, this.canvasWidth, this.canvasHeight);
    //设置画笔颜色
    this.canvasContext.fillStyle = "#111111";
    //绘制边框需要粗一点
    this.canvasContext.lineWidth = 4;
    //绘制棋盘边框
    this.canvasContext.beginPath();
    this.canvasContext.moveTo(2, 2);
    this.canvasContext.lineTo(2, this.canvasWidth - 2);
    this.canvasContext.lineTo(this.canvasHeight - 2, this.canvasWidth - 2);
    this.canvasContext.lineTo(this.canvasHeight - 2, 2);
    this.canvasContext.closePath();
    this.canvasContext.stroke();
    //绘制分割线用1px宽度足矣
    this.canvasContext.lineWidth = 1;
    for (let i = 0; i < this.size; i++) {
      let line_w = Math.round((i + 1) * this.latticeWidth);
      let line_h = Math.round((i + 1) * this.latticeHeight);
      //绘制一条竖线
      this.canvasContext.beginPath();
      this.canvasContext.moveTo(line_w, 0);
      this.canvasContext.lineTo(line_w, this.canvasHeight);
      this.canvasContext.closePath();
      this.canvasContext.stroke();
      //绘制一条横线
      this.canvasContext.beginPath();
      this.canvasContext.moveTo(0, line_h);
      this.canvasContext.lineTo(this.canvasWidth, line_h);
      this.canvasContext.closePath();
      this.canvasContext.stroke();
    }
    for (let i = 0; i < this.stepStack.length; i++) {
      let [x, y, who] = this.stepStack[i];
      //开始绘制棋子
      if (who) {
        //绘制一个黑子
        this.canvasContext.fillStyle = "#000000";
        this.canvasContext.beginPath();
        this.canvasContext.arc(
          (x + 1) * this.latticeWidth,
          (y + 1) * this.latticeHeight,
          this.latticeWidth * 0.4,
          0,
          2 * Math.PI
        );
        this.canvasContext.closePath();
        this.canvasContext.fill();
      } else {
        //绘制一个白子
        this.canvasContext.fillStyle = "#FFFFFF";
        this.canvasContext.beginPath();
        this.canvasContext.arc(
          (x + 1) * this.latticeWidth,
          (y + 1) * this.latticeHeight,
          this.latticeWidth * 0.4,
          0,
          2 * Math.PI
        );
        this.canvasContext.closePath();
        this.canvasContext.fill();
      }
    }
  }
  /**
   * dropChessman
   * @description:
   *   下一颗棋子
   * @param: {number} x 横坐标
   * @param: {number} y 纵坐标
   * @return: {void}
   */
  dropChessman(x: number, y: number): void {
    //游戏未开始
    if (this.playing === false) return;
    //当前位置已经下过棋子了
    if (this.matrix[x][y] !== 0) return;
    //每一步棋入栈
    this.stepStack.push([x, y, this.whoDropping]);
    //往矩阵中写入棋子信息
    this.matrix[x][y] = this.whoDropping ? 1 : 2;
    //绘制最新棋盘
    this.drawChessboard();
    //判断赢家
    switch (this.whoIsWinner()) {
      case 1:
        this.playing = false;
        setTimeout(() => {
          alert("黑子赢了");
        }, 200);
        break;
      case 2:
        this.playing = false;
        setTimeout(() => {
          alert("白子赢了");
        }, 200);
        break;

      default:
        break;
    }
    //切换下棋者
    this.whoDropping = !this.whoDropping;
  }
  /**
   * whoIsWinner
   * @description:
   *   判断谁是赢家
   * @return: {number}0没有赢家 1黑子赢 2白子赢
   */
  whoIsWinner(): number {
    for (let i = 2; i < this.size - 2; i++) {
      for (let j = 2; j < this.size - 2; j++) {
        if (this.matrix[i][j] > 0) {
          if (
            (this.matrix[i][j] === this.matrix[i - 2][j - 2] &&
              this.matrix[i][j] === this.matrix[i - 1][j - 1] &&
              this.matrix[i][j] === this.matrix[i + 2][j + 2] &&
              this.matrix[i][j] === this.matrix[i + 1][j + 1]) ||
            (this.matrix[i][j] === this.matrix[i + 1][j - 1] &&
              this.matrix[i][j] === this.matrix[i + 2][j - 2] &&
              this.matrix[i][j] === this.matrix[i - 1][j + 1] &&
              this.matrix[i][j] === this.matrix[i - 2][j + 2]) ||
            (this.matrix[i][j] === this.matrix[i - 2][j] &&
              this.matrix[i][j] === this.matrix[i - 1][j] &&
              this.matrix[i][j] === this.matrix[i + 2][j] &&
              this.matrix[i][j] === this.matrix[i + 1][j]) ||
            (this.matrix[i][j] === this.matrix[i][j - 2] &&
              this.matrix[i][j] === this.matrix[i][j - 1] &&
              this.matrix[i][j] === this.matrix[i][j + 2] &&
              this.matrix[i][j] === this.matrix[i][j + 1])
          ) {
            return this.matrix[i][j];
          }
        }
      }
    }
    return 0;
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.mapDiv {
  margin: 12px auto;
  text-align: center;
}
.map {
  box-shadow: none;
  border: 10px solid rgb(194, 149, 107); /*rgb(214,159,64);*/
  width: calc(75vh);
  height: calc(75vh);
}
@media screen and (max-width: 600px) {
  .map {
    width: calc(90vw);
    height: calc(90vw);
  }
  .mapDiv {
    margin: 30px auto;
    margin-bottom: 0;
    text-align: center;
  }
}
.button-group {
  width: calc(80vh);
  margin: auto;
}
.button {
  user-select: none;
  cursor: pointer;
  background-color: rgb(194, 149, 107);
  margin-left: calc(2.5vh);
  margin-right: calc(2.5vh);
  margin-top: 12px;
  width: calc(35vh);
  height: 50px;
  line-height: 50px;
  box-shadow: 4px 4px 4px gray;
  border-radius: 10px;
  float: left;
}
.button:active {
  box-shadow: 1px 1px 1px gray;
  transform: translateY(4px);
}
/*手机等设备风格*/
@media screen and (max-width: 600px) {
  .button-group {
    width: 100%;
  }
  .button {
    margin: auto;
    margin-top: 12px;
    width: 100%;
    font-weight: bold;
    justify-content: center;
  }
}
</style>
