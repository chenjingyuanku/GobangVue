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

enum PointColor {
  Empty = 0,
  Black,
  White
}
enum ScanDir {
  LT = 0,
  T,
  RT,
  L,
  R,
  LB,
  B,
  RB
}
const Who = {
  Black: true,
  White: false
};
class dirType {
  //区间内我方棋子总个数，遇到连续2空子停止计数
  myCount: number = 0;
  //区间内我方连续棋子个数
  myConsecutiveCount: number = 0;
  //一个方向的扫描距离，最大四格，到敌方棋子、棋盘边缘截止
  mySearchArea: number = 0;
  //区间内敌方棋子总个数，遇到连续2空子停止计数
  enemyCount: number = 0;
  //区间内敌方连续棋子个数
  enemyConsecutiveCount: number = 0;
  //一个方向的扫描距离，最大四格，到我方棋子、棋盘边缘截止
  enemySearchArea: number = 0;
  //己方颜色
  private myColor: PointColor = PointColor.Black;
  private enemyColor: PointColor = PointColor.White;
  private dir!: ScanDir;
  private x!: number;
  private y!: number;
  private size!: number;
  private matrix!: Array<Array<number>>;

  private myConsecutiveFlag: boolean = true;
  private enemyConsecutiveFlag: boolean = true;
  private myScanFlag: boolean = true;
  private enemyScanFlag: boolean = true;
  private consecutiveEmptyCountFlag: boolean = true;

  private previousColor: PointColor = PointColor.Empty;
  constructor(
    matrix: Array<Array<number>>,
    x: number,
    y: number,
    color: PointColor,
    dir: ScanDir,
    size: number
  ) {
    this.myColor = color;
    this.dir = dir;
    this.size = size;
    this.matrix = matrix;
    this.myConsecutiveFlag = true;
    this.enemyConsecutiveFlag = true;
    this.myScanFlag = true;
    this.enemyScanFlag = true;
    this.consecutiveEmptyCountFlag = true;
    this.x = x;
    this.y = y;
    if (this.myColor !== PointColor.Black) this.enemyColor = PointColor.Black;
    this.scan();
  }
  /**
   * count
   * @description:
   *   统计扫描到的点并保存
   * @param: {number} x 当前横坐标
   * @param: {number} y 当前纵坐标
   * @param: {number} c 当前扫到第几个
   * @return: {void}
   */
  count(x: number, y: number, c: number) {
    if (c > 0) {
      if (
        this.matrix[x][y] === PointColor.Empty &&
        this.previousColor === PointColor.Empty
      ) {
        this.consecutiveEmptyCountFlag = false;
      }
    }
    //扫到己方棋子，中断敌方棋子扫描
    if (this.matrix[x][y] === this.myColor) {
      this.enemyScanFlag = false;
    }
    //扫到敌方棋子，中断己方棋子扫描
    else if (this.matrix[x][y] === this.enemyColor) {
      this.myScanFlag = false;
    }
    //扫到空子，连续标志设置为false，对双方都一样
    else {
      this.myConsecutiveFlag = false;
      this.enemyConsecutiveFlag = false;
    }
    //如果允许扫描
    if (this.myScanFlag) {
      //扫描距离增加
      this.mySearchArea++;
      //对己方棋子计数
      if (
        this.matrix[x][y] === this.myColor &&
        this.consecutiveEmptyCountFlag
      ) {
        this.myCount++;
      }
      //对连续己方棋子计数
      if (this.myConsecutiveFlag) {
        this.myConsecutiveCount++;
      }
    }
    if (this.enemyScanFlag) {
      this.enemySearchArea++;
      if (
        this.matrix[x][y] === this.enemyColor &&
        this.consecutiveEmptyCountFlag
      ) {
        this.enemyCount++;
      }
      if (this.enemyConsecutiveFlag) {
        this.enemyConsecutiveCount++;
      }
    }
    this.previousColor = this.matrix[x][y];
  }
  /**
   * scan
   * @description:
   *   对某个方向进行扫描
   * @return: {void}
   */
  scan() {
    switch (this.dir) {
      case ScanDir.LT:
        for (
          let i = this.x - 1, j = this.y - 1, c = 0;
          i > 0 && j > 0 && c < 4;
          i--, j--, c++
        ) {
          this.count(i, j, c);
        }
        break;
      case ScanDir.T:
        for (let i = this.x, j = this.y - 1, c = 0; j > 0 && c < 4; j--, c++) {
          this.count(i, j, c);
        }
        break;
      case ScanDir.RT:
        for (
          let i = this.x + 1, j = this.y - 1, c = 0;
          i < this.size && j > 0 && c < 4;
          i++, j--, c++
        ) {
          this.count(i, j, c);
        }
        break;
      case ScanDir.L:
        for (let i = this.x - 1, j = this.y, c = 0; i > 0 && c < 4; i--, c++) {
          this.count(i, j, c);
        }
        break;
      case ScanDir.R:
        for (
          let i = this.x + 1, j = this.y, c = 0;
          i < this.size && c < 4;
          i++, c++
        ) {
          this.count(i, j, c);
        }
        break;
      case ScanDir.LB:
        for (
          let i = this.x - 1, j = this.y + 1, c = 0;
          i > 0 && j < this.size && c < 4;
          i--, j++, c++
        ) {
          this.count(i, j, c);
        }
        break;
      case ScanDir.B:
        for (
          let i = this.x, j = this.y + 1, c = 0;
          j < this.size && c < 4;
          j++, c++
        ) {
          this.count(i, j, c);
        }
        break;
      case ScanDir.RB:
        for (
          let i = this.x + 1, j = this.y + 1, c = 0;
          i < this.size && j < this.size && c < 4;
          i++, j++, c++
        ) {
          this.count(i, j, c);
        }
        break;
      default:
        break;
    }
  }
  reset() {
    this.myCount = 0;
    this.myConsecutiveCount = 0;
    this.mySearchArea = 0;
    this.enemyCount = 0;
    this.enemyConsecutiveCount = 0;
    this.enemySearchArea = 0;
    this.myConsecutiveFlag = true;
    this.enemyConsecutiveFlag = true;
    this.myScanFlag = true;
    this.enemyScanFlag = true;
  }
}
class emptyPoint {
  private dirLeftTop!: dirType;
  private dirTop!: dirType;
  private dirRightTop!: dirType;
  private dirLeft!: dirType;
  private dirRight!: dirType;
  private dirLeftBottom!: dirType;
  private dirBottom!: dirType;
  private dirRightBottom!: dirType;
  private myColor: PointColor = PointColor.Black;
  score: number = 0;
  private positionScore: number = 0;
  private matrix!: Array<Array<number>>;
  constructor(
    matrix: Array<Array<number>>,
    x: number,
    y: number,
    myColor: PointColor,
    size: number
  ) {
    this.myColor = myColor; //保存我方棋子颜色
    this.dirLeftTop = new dirType(matrix, x, y, myColor, ScanDir.LT, size);
    this.dirTop = new dirType(matrix, x, y, myColor, ScanDir.T, size);
    this.dirRightTop = new dirType(matrix, x, y, myColor, ScanDir.RT, size);
    this.dirLeft = new dirType(matrix, x, y, myColor, ScanDir.L, size);
    this.dirRight = new dirType(matrix, x, y, myColor, ScanDir.R, size);
    this.dirLeftBottom = new dirType(matrix, x, y, myColor, ScanDir.LB, size);
    this.dirBottom = new dirType(matrix, x, y, myColor, ScanDir.B, size);
    this.dirRightBottom = new dirType(matrix, x, y, myColor, ScanDir.RB, size);
    this.matrix = matrix;
    this.positionScore =
      (Math.sqrt((size / 2) * (size / 2) * 2) -
        Math.sqrt(
          (x - size / 2) * (x - size / 2) + (y - size / 2) * (y - size / 2)
        )) *
        3 +
      Math.random() * 5;
    this.score = this.positionScore + this.calcScoreSum();
  }
  /**
   * calcMine
   * @description:
   *   计算己方得分
   * @param: {dirType} dirA 方向A
   * @param: {dirType} dirB 方向B，在A的对面
   * @return: {number} 返回计算出来的分值
   */
  private calcMine(dirA: dirType, dirB: dirType): number {
    let closeSides: number = 0;
    let validArea: number = 0;
    let myCountSum: number = 0;
    myCountSum = dirA.myCount + dirB.myCount;
    validArea = dirA.mySearchArea + dirB.mySearchArea;
    if (validArea < 4) {
      return 0; //TODO 一个很小的值
    }
    if (dirA.mySearchArea === dirA.myConsecutiveCount) {
      closeSides++;
    }
    if (dirB.mySearchArea === dirB.myConsecutiveCount) {
      closeSides++;
    }
    switch (dirA.myConsecutiveCount + dirB.myConsecutiveCount) {
      case 0:
        //以下是当前方向上分散棋子个数，间隔不超过1
        switch (myCountSum) {
          case 0:
            return 0;
          case 1:
            return 100;
          case 2:
            return 200;
          case 3:
            return 400;
          default:
            return 2000;
        }
      case 1:
        //2连，2头空
        if (closeSides === 0) {
          //活跳三
          if (myCountSum === 2) {
            return 4000;
          }
          //冲四
          if (myCountSum >= 3) {
            return 5000;
          }
          //普通连2
          return 1000;
        } //眠二
        else {
          return 100;
        }
      case 2:
        //3连，2头空
        if (closeSides === 0) {
          //冲四
          if (myCountSum >= 3) {
            return 5000;
          }
          //普通连3
          return 4000;
        } //眠三
        else {
          return 1000;
        }
      case 3:
        //活四
        if (closeSides === 0) {
          return 30000;
        } //冲四
        else {
          return 5000;
        }
      default:
        //五连子了，返回最大得分
        return 50 * 10000;
    }
  }

  /**
   * calcEnemy
   * @description:
   *   计算敌方得分
   * @param: {dirType} dirA 方向A
   * @param: {dirType} dirB 方向B，在A的对面
   * @return: {number} 返回计算出来的分值
   */
  private calcEnemy(dirA: dirType, dirB: dirType): number {
    let closeSides: number = 0;
    let validArea: number = 0;
    let enemyCountSum: number = 0;
    enemyCountSum = dirA.enemyCount + dirB.enemyCount;
    validArea = dirA.enemySearchArea + dirB.enemySearchArea;
    if (validArea < 4) {
      return 0; //TODO 一个很小的值
    }
    if (dirA.enemySearchArea === dirA.enemyConsecutiveCount) {
      closeSides++;
    }
    if (dirB.enemySearchArea === dirB.enemyConsecutiveCount) {
      closeSides++;
    }
    switch (dirA.enemyConsecutiveCount + dirB.enemyConsecutiveCount) {
      case 0:
        //以下是当前方向上分散棋子个数，间隔不超过1
        switch (enemyCountSum) {
          case 0:
            return 0;
          case 1:
            return 100;
          case 2:
            return 200;
          case 3:
            return 400;
          default:
            return 2000;
        }
      case 1:
        //2连，2头空
        if (closeSides === 0) {
          //活跳三
          if (enemyCountSum === 2) {
            return 3000;
          }
          //冲四
          if (enemyCountSum >= 3) {
            return 4000;
          }
          //普通连2
          return 1000;
        } //眠二
        else {
          return 100;
        }
      case 2:
        //3连，2头空
        if (closeSides === 0) {
          //冲四
          if (enemyCountSum >= 3) {
            return 4000;
          }
          //普通连3
          return 3000;
        } //眠三
        else {
          return 1000;
        }
      case 3:
        //活四
        if (closeSides === 0) {
          return 21000;
        } //冲四
        else {
          return 5000;
        }
      default:
        //敌方五连子了，返回仅次于己方五连子的最大得分
        return 10 * 10000;
    }
  }
  /**
   * calcLeftToRightScore
   * @description:
   *   计算左到右方向上的分值
   * @return: {number} 返回计算出来的分值，己方和敌方分数之和
   */
  private calcLeftToRightScore(): number {
    return (
      this.calcMine(this.dirLeft, this.dirRight) +
      this.calcEnemy(this.dirLeft, this.dirRight)
    );
  }
  /**
   * calcLeftTopToRightBottomScore
   * @description:
   *   计算左上到右下方向上的分值
   * @return: {number} 返回计算出来的分值，己方和敌方分数之和
   */
  private calcLeftTopToRightBottomScore(): number {
    return (
      this.calcMine(this.dirLeftTop, this.dirRightBottom) +
      this.calcEnemy(this.dirLeftTop, this.dirRightBottom)
    );
  }
  /**
   * calcTopToBottomScore
   * @description:
   *   计算上到下方向上的分值
   * @return: {number} 返回计算出来的分值，己方和敌方分数之和
   */
  private calcTopToBottomScore(): number {
    return (
      this.calcMine(this.dirTop, this.dirBottom) +
      this.calcEnemy(this.dirTop, this.dirBottom)
    );
  }
  /**
   * calcRightTopToLeftBottomScore
   * @description:
   *   计算右上到左下方向上的分值
   * @return: {number} 返回计算出来的分值，己方和敌方分数之和
   */
  private calcRightTopToLeftBottomScore(): number {
    return (
      this.calcMine(this.dirRightTop, this.dirLeftBottom) +
      this.calcEnemy(this.dirRightTop, this.dirLeftBottom)
    );
  }
  /**
   * calcScoreSum
   * @description:
   *   计算所有方向的分值综合
   * @return: {number} 返回计算出来的分值，8个方向之和
   */
  private calcScoreSum(): number {
    let scoreSum =
      this.calcLeftToRightScore() +
      this.calcLeftTopToRightBottomScore() +
      this.calcTopToBottomScore() +
      this.calcRightTopToLeftBottomScore();
    return scoreSum;
  }
}
@Component
export default class Map extends Vue {
  canvasContext!: CanvasRenderingContext2D; //画布对象
  canvasWidth: number = 200; //画布宽度
  canvasHeight: number = 200; //画布高度
  latticeWidth: number = 200; //小格子宽度
  latticeHeight: number = 200; //小格子高度
  gameMode: number = 0; //0-人人对战 1-人机对战
  whoDropping: boolean = Who.Black; //true黑子 false白子
  playing: boolean = true; //游戏进行状态
  stepStack: Array<[number, number, boolean]> = []; //记步栈
  matrix: Array<Array<number>> = new Array<Array<PointColor>>(); //棋盘矩阵0空1黑2白
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
        this.matrix[i].push(PointColor.Empty);
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
    //将点击的坐标点转换成棋盘矩阵的坐标
    let x: number = Math.round((event.offsetX * 1.0) / this.latticeWidth);
    let y: number = Math.round((event.offsetY * 1.0) / this.latticeHeight);
    //超出棋盘范围了
    if (x < 1 || x > this.size || y < 1 || y > this.size) return;
    // 人机人白没轮到自己，return
    if (this.gameMode === 1 && this.whoDropping === Who.Black) {
      return;
    }
    // 人机人黑没轮到自己，return
    if (this.gameMode === 2 && this.whoDropping === Who.White) {
      return;
    }
    if (!this.playing) {
      alert("请点击开始游戏按钮！");
      return;
    }
    this.dropChessman(x - 1, y - 1);
  }

  /**
   * calcPlacementCoordinate
   * @description:
   *   计算落子点坐标
   * @param: {Array<Array<number>>} map 上一次模拟后的情况
   * @param: {PointColor} myColor 1黑 2白 模拟中的颜色
   * @param: {PointColor} sourceColor 1黑 2白 实际的颜色
   * @param: {number} depth 递归模拟深度
   * @param: {number} surplusEmptyPointsCount 剩余空点个数
   * @return: {object} {x:横坐标,y:纵坐标}
   */
  calcPlacementCoordinate(
    map: Array<Array<number>>,
    myColor: PointColor,
    sourceColor: PointColor,
    depth: number,
    surplusEmptyPointsCount: number
  ): { x: number; y: number; score: number } {
    let highScorePointList: Array<{ x: number; y: number; score: number }> = [
      {
        x: -1,
        y: -1,
        score: -1
      },
      {
        x: -2,
        y: -2,
        score: -2
      }
    ];
    let enemyColor =
      myColor === PointColor.Black ? PointColor.White : PointColor.Black;
    //创建新地图
    let newMap1: Array<Array<number>> = [];
    let newMap2: Array<Array<number>> = [];
    for (let i = 0; i < this.size; i++) {
      newMap1.push([]);
      newMap2.push([]);
      for (let j = 0; j < this.size; j++) {
        newMap1[i].push(map[i][j]);
        newMap2[i].push(map[i][j]);
        if (map[i][j] !== PointColor.Empty) {
          continue;
        }
        let point = new emptyPoint(map, i, j, myColor, this.size);
        if (point.score > highScorePointList[0].score) {
          highScorePointList.pop();
          highScorePointList.unshift({ x: i, y: j, score: point.score });
        } else if (point.score > highScorePointList[0].score) {
          highScorePointList.pop();
          highScorePointList.push({ x: i, y: j, score: point.score });
        }
      }
    }
    //TODO 深度为0或者剩余空间不足或者有一方胜出
    if (depth === 0 || surplusEmptyPointsCount <= 1) {
      return highScorePointList[0];
    } else {
      newMap1[highScorePointList[0].x][highScorePointList[0].y] = myColor;
      newMap2[highScorePointList[1].x][highScorePointList[1].y] = myColor;
      let point1 = this.calcPlacementCoordinate(
        newMap1,
        enemyColor,
        sourceColor,
        depth - 1,
        surplusEmptyPointsCount - 1
      );
      let point2 = this.calcPlacementCoordinate(
        newMap2,
        enemyColor,
        sourceColor,
        depth - 1,
        surplusEmptyPointsCount - 1
      );
      if (sourceColor === enemyColor) {
        highScorePointList[0].score += point1.score;
        highScorePointList[1].score += point2.score;
      } else {
        highScorePointList[0].score -= point1.score;
        highScorePointList[1].score -= point2.score;
      }
      if (highScorePointList[0].score > highScorePointList[1].score) {
        return highScorePointList[0];
      } else {
        return highScorePointList[1];
      }
    }
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
    if (this.gameMode === 1 && this.whoDropping === Who.Black) {
      let pos = this.calcPlacementCoordinate(
        this.matrix,
        PointColor.Black,
        PointColor.Black,
        0,
        this.size * this.size - this.stepStack.length
      );
      this.dropChessman(pos.x, pos.y);
    }
    //人机人先且当前轮到白子
    if (this.gameMode === 2 && this.whoDropping === Who.White) {
      let pos = this.calcPlacementCoordinate(
        this.matrix,
        PointColor.White,
        PointColor.White,
        0,
        this.size * this.size - this.stepStack.length
      );
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
    //状态切换为游戏中
    this.playing = true;
    if (this.gameMode === 0) {
      //栈空了，return
      if (this.stepStack.length < 1) return;
      //最近一步棋出栈
      let [x, y, who] = this.stepStack.pop() as [number, number, boolean];
      //清除矩阵对应位置的记录
      this.matrix[x][y] = PointColor.Empty;
      //切换棋手
      this.whoDropping = who;
    } else if (
      (this.gameMode === 1 && this.whoDropping === Who.White) ||
      (this.gameMode === 2 && this.whoDropping === Who.Black)
    ) {
      //人机模式，人执棋可以悔棋
      //栈数据不足，return
      if (this.stepStack.length < 2) return;
      //最近2步棋出栈
      let [x1, y1, who1] = this.stepStack.pop() as [number, number, boolean];
      let [x2, y2, who2] = this.stepStack.pop() as [number, number, boolean];
      //清除矩阵对应位置的记录
      this.matrix[x1][y1] = PointColor.Empty;
      this.matrix[x2][y2] = PointColor.Empty;
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
        this.matrix[i][j] = PointColor.Empty;
      }
    }
    //清空落子记录栈
    this.stepStack.length = 0;
    //重新绘制棋盘
    this.drawChessboard();
    //落子权回到黑子
    this.whoDropping = Who.Black;
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
    this.canvasContext!.fillStyle = "RGB(194,149,107)";
    //绘制矩形背景
    this.canvasContext!.fillRect(0, 0, this.canvasWidth, this.canvasHeight);
    //设置画笔颜色
    this.canvasContext.fillStyle = "#111111";
    //绘制边框需要粗一点
    this.canvasContext.lineWidth = 6;
    //设置文字居中
    this.canvasContext.textAlign = "center";
    this.canvasContext.textBaseline = "middle";
    //设置字体和大小
    this.canvasContext.font = "14px Arial";
    //绘制棋盘边框
    this.canvasContext.beginPath();
    this.canvasContext.moveTo(this.latticeWidth, this.latticeHeight);
    this.canvasContext.lineTo(
      this.latticeWidth,
      this.canvasHeight - this.latticeHeight
    );
    this.canvasContext.lineTo(
      this.canvasWidth - this.latticeWidth,
      this.canvasHeight - this.latticeHeight
    );
    this.canvasContext.lineTo(
      this.canvasWidth - this.latticeWidth,
      this.latticeHeight
    );
    this.canvasContext.closePath();
    this.canvasContext.stroke();
    //绘制分割线用1px宽度足矣
    this.canvasContext.lineWidth = 1;
    for (let i = 0; i < this.size; i++) {
      let line_w = Math.round((i + 1) * this.latticeWidth);
      let line_h = Math.round((i + 1) * this.latticeHeight);
      //绘制一条竖线
      this.canvasContext.beginPath();
      this.canvasContext.moveTo(line_w, this.latticeHeight);
      this.canvasContext.lineTo(line_w, this.canvasHeight - this.latticeHeight);
      this.canvasContext.closePath();
      this.canvasContext.stroke();
      //绘制一条横线
      this.canvasContext.beginPath();
      this.canvasContext.moveTo(this.latticeWidth, line_h);
      this.canvasContext.lineTo(this.canvasWidth - this.latticeWidth, line_h);
      this.canvasContext.closePath();
      this.canvasContext.stroke();
    }
    //画五个黑点
    this.canvasContext.beginPath();
    this.canvasContext.arc(
      4 * this.latticeWidth,
      4 * this.latticeHeight,
      this.latticeWidth * 0.15,
      0,
      2 * Math.PI
    );
    this.canvasContext.arc(
      12 * this.latticeWidth,
      4 * this.latticeHeight,
      this.latticeWidth * 0.15,
      0,
      2 * Math.PI
    );
    this.canvasContext.fill();
    this.canvasContext.beginPath();
    this.canvasContext.arc(
      4 * this.latticeWidth,
      12 * this.latticeHeight,
      this.latticeWidth * 0.15,
      0,
      2 * Math.PI
    );
    this.canvasContext.arc(
      12 * this.latticeWidth,
      12 * this.latticeHeight,
      this.latticeWidth * 0.15,
      0,
      2 * Math.PI
    );
    this.canvasContext.fill();
    this.canvasContext.beginPath();
    this.canvasContext.arc(
      8 * this.latticeWidth,
      8 * this.latticeHeight,
      this.latticeWidth * 0.15,
      0,
      2 * Math.PI
    );
    this.canvasContext.fill();
    //开始绘制棋子
    for (let i = 0; i < this.stepStack.length; i++) {
      let [x, y, who] = this.stepStack[i];
      if (who) {
        //绘制一个黑子
        this.canvasContext.fillStyle = "#000000";
        this.canvasContext.beginPath();
        this.canvasContext.arc(
          (x + 1) * this.latticeWidth,
          (y + 1) * this.latticeHeight,
          this.latticeWidth * 0.42,
          0,
          2 * Math.PI
        );
        this.canvasContext.closePath();
        this.canvasContext.fill();
        this.canvasContext.fillStyle = "#FFFFFF";
      } else {
        //绘制一个白子
        this.canvasContext.fillStyle = "#FFFFFF";
        this.canvasContext.beginPath();
        this.canvasContext.arc(
          (x + 1) * this.latticeWidth,
          (y + 1) * this.latticeHeight,
          this.latticeWidth * 0.42,
          0,
          2 * Math.PI
        );
        this.canvasContext.closePath();
        this.canvasContext.fill();
        this.canvasContext.fillStyle = "#000000";
      }
      //绘制棋子编号
      this.canvasContext.fillText(
        `${i + 1}`,
        (x + 1) * this.latticeWidth,
        (y + 1) * this.latticeHeight,
        this.latticeWidth * 0.7
      );
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
    if (this.matrix[x][y] !== PointColor.Empty) return;
    //每一步棋入栈
    this.stepStack.push([x, y, this.whoDropping]);
    //往矩阵中写入棋子信息
    this.matrix[x][y] = this.whoDropping ? PointColor.Black : PointColor.White;
    //绘制最新棋盘
    this.drawChessboard();
    //判断赢家
    switch (this.whoIsWinner()) {
      case PointColor.Black:
        this.playing = false;
        setTimeout(() => {
          alert("黑子赢了");
        }, 200);
        break;
      case PointColor.White:
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
  whoIsWinner(): PointColor {
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
    return PointColor.Empty;
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
  border-radius: 10px;
  width: calc(75vh);
  height: calc(75vh);
}
@media screen and (max-width: 600px) {
  .map {
    margin: 0;
    width: calc(95vw);
    height: calc(95vw);
  }
  .mapDiv {
    margin: 10px auto;
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
    margin-top: 10px;
    width: 100%;
    font-weight: bold;
    justify-content: center;
  }
}
</style>
