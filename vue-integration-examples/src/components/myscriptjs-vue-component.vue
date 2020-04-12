<template>
  <div>
    <h1>{{ msg }}</h1>
    <h3>{{ langhuaHandwritingMsg }}</h3>
    <div class="writing-container" touch-action="none" ref="editor" id="editor"></div>
    <div id="result" ref="result"></div>
    <div id="message" ref="message">
      <svg id="hanSvg" ref="hanSvg" version="1.1" xmlns="http://www.w3.org/2000/svg" width="200px" height="200px" viewBox="0 0 254 254">
        <path class="migrid" d="M0 0 L254 0 L254 254 L0 254 Z"/>
        <path class="migriddot" d="M0 0 L254 254 M0 254 L254 0 M127 0 L 127 254 M 0 127 L254 127"/>
      </svg>
      <div id="errorMsg" ref="errorMsg"></div>
    </div>
  </div>
</template>

<script>
import * as MyScriptJS from 'myscript/src/myscript';
import anime from 'animejs';

export default {
  name: 'MyScriptJSVueComponent',
  data() {
    return {
      msg: 'Basic example of vue integration of MyScriptJS',
      langhuaHandwritingMsg: '朗华汉字笔画/笔顺/优美度检查',
      strokes: undefined,
      hanChar: undefined,
      runningWaitForIdle: undefined,
      idleStart: undefined,
      idleEnd: undefined,
      myscriptBasicCheckUrl: 'http://127.0.0.1:8080/handwriting/MyScriptBasicCheck',
      idleTimeout: 2000,
    };
  },
  created() {
    // eslint-disable-next-line
    console.log('Created' + this.$refs['editor']);
  },
  mounted() {
    // Fired every second, should always be true
    // eslint-disable-next-line
    console.log('Mounted ' + this.$refs['editor']);

    this.$refs.editor.addEventListener('exported', this.exportStrokes);

    this.$refs.editor.addEventListener('idle', this.processIdle);

    MyScriptJS.register(this.$refs.editor, {
      recognitionParams: {
        type: 'TEXT',
        protocol: 'WEBSOCKET',
        apiVersion: 'V4',
        server: {
          scheme: 'https',
          host: 'webdemoapi.myscript.com',
          applicationKey: 'f1355ec8-c74a-4da9-8d63-691ab05952eb',
          hmacKey: '752acf37-5a45-481b-9361-fcb32cd7f6a1',
        },
        v4: {
          export: {
            jiix: {
              strokes: true,
            },
          },
          lang: 'zh_CN',
        },
      },
    });
  },
  methods: {
    exportStrokes(event) {
      if (
        event &&
        event.detail &&
        event.detail.exports &&
        'application/vnd.myscript.jiix' in event.detail.exports
      ) {
        const jiix = JSON.parse(
          event.detail.exports['application/vnd.myscript.jiix'],
        );
        if (
          jiix.words &&
          jiix.words['0'] &&
          jiix.words['0'].label &&
          jiix.words['1'] === undefined
        ) {
          this.hanChar = jiix.words['0'].label;
          this.strokes = jiix.words['0'].items;
          if (!this.runningWaitForIdle) {
            this.$refs.editor.editor.waitForIdle();
            this.runningWaitForIdle = true;
          }
        } else {
          this.hanChar = undefined;
          this.strokes = undefined;
        }
      }
    },
    processIdle(event) {
      if (!event.detail.idle) {
        this.idleStart = event.timeStamp;
      } else {
        this.idleEnd = event.timeStamp;
        if (this.idleEnd - this.idleStart < this.idleTimeout) {
          window.setTimeout(() => {
            this.$refs.editor.editor.waitForIdle();
          }, (this.idleTimeout + 100 + this.idleStart) - this.idleEnd);
        } else {
          if (!this.runningWaitForIdle) {
            if (this.hanChar !== undefined && this.strokes !== undefined) {
              if (this.checkChinese(this.hanChar)) {
                this.getStrokeEvalResult(
                  this.hanChar,
                  JSON.stringify(this.strokes),
                );
              } else {
                this.$refs.errorMsg.innerHTML = `【${this.hanChar}】不是一个汉字，无法做笔顺和优美度检查。`;
              }
              this.hanChar = undefined;
              this.strokes = undefined;
            }
            this.$refs.editor.editor.clear();
          }
          this.runningWaitForIdle = false;
        }
      }
    },
    checkChinese(val) {
      const reg = new RegExp('[\\u4E00-\\u9FFF]+', 'g');
      return reg.test(val);
    },
    getStrokeEvalResult(hanzi, myscriptStrokes) {
      const params = {
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
        },
        body: `han=${hanzi}&strokes=${myscriptStrokes}`,
        method: 'POST',
      };
      fetch(this.myscriptBasicCheckUrl, params)
        .then(response => response.json())
        .then((value) => {
          if (value.result === 'error' && value.message.length > 0) {
            this.$refs.errorMsg.innerHTML = `【${value.han}】${value.message}`;
            this.$refs.message.style.display = 'block';
          } else {
            this.$refs.errorMsg.innerHTML = '';
            this.$refs.message.style.display = 'none';
          }
          if (value.result === 'success' && value.score > 0) {
            let score = Math.round(value.score / 2);
            if (score >= 95) {
              score = 100;
            }
            this.$refs.result.innerHTML = this.$refs.result.innerHTML
              ? `${this.$refs.result.innerHTML}, ${value.han}:${value.message}/${score}`
              : `${value.han}:${value.message}/${score}`;
          }
          if (value.stdStrokes !== undefined) {
            this.animateStdHanStroke(value.stdStrokes,
              value.wrongStrokeDirectionAt,
              value.wrongStrokeAt,
              value.stdStrokeNum);
          }
        })
        .catch((error) => {
          window.console.log(error);
        });
    },
    animateStdHanStroke(stdStrokes, wrongStrokeDirectionAt, wrongStrokeAt, stdStrokeNum) {
      const hanSvg = document.getElementById('hanSvg');
      let newSvgContent = hanSvg.innerHTML;
      for (let j = 0; j < stdStrokes.length; j += 1) {
        const strokes = stdStrokes[j];
        let oneStrokePath = '';
        const isBezier = this.useBezierCurve(strokes.x, strokes.y);
        if (isBezier) {
          switch (strokes.x.length) {
            case 3:
              // M x0,y0 Q x1,y1 x2,y2
              oneStrokePath = `M${strokes.x[0]},${strokes.y[0]}`;
              oneStrokePath += ` Q${strokes.x[1]},${strokes.y[1]} ${strokes.x[2]},${strokes.y[2]}`;
              break;
            case 4:
              // M x0,y0 C x1,y1 x2,y2 x3,y3
              oneStrokePath = `M${strokes.x[0]},${strokes.y[0]}`;
              oneStrokePath += ` C${strokes.x[1]},${strokes.y[1]} ${strokes.x[2]},${strokes.y[2]} ${strokes.x[3]},${strokes.y[3]}`;
              break;
            case 5:
              // M x0,y0 Q x1,y1 x2,y2 T x4,y4
              oneStrokePath = `M${strokes.x[0]},${strokes.y[0]}`;
              oneStrokePath += ` Q${strokes.x[1]},${strokes.y[1]} ${strokes.x[2]},${strokes.y[2]}`;
              oneStrokePath += ` T${strokes.x[4]},${strokes.y[4]}`;
              break;
            case 6:
              // M x0,y0 C x1,y1 x2,y2 x3,y3 S x4,y4 x5,y5
              oneStrokePath = `M${strokes.x[0]},${strokes.y[0]}`;
              oneStrokePath += ` C${strokes.x[1]},${strokes.y[1]} ${strokes.x[2]},${strokes.y[2]} ${strokes.x[3]},${strokes.y[3]}`;
              oneStrokePath += ` S${strokes.x[4]},${strokes.y[4]} ${strokes.x[5]},${strokes.y[5]}`;
              break;
            case 7:
              // M x0,y0 C x1,y1 x2,y2 x3,y3 S x5,y5 x6,y6
              oneStrokePath = `M${strokes.x[0]},${strokes.y[0]}`;
              oneStrokePath += ` C${strokes.x[1]},${strokes.y[1]} ${strokes.x[2]},${strokes.y[2]} ${strokes.x[3]},${strokes.y[3]}`;
              oneStrokePath += ` S${strokes.x[5]},${strokes.y[5]} ${strokes.x[6]},${strokes.y[6]}`;
              break;
            default:
              for (let i = 0; i < strokes.x.length; i += 1) {
                const x = strokes.x[i];
                const y = strokes.y[i];
                if (i === 0) {
                  oneStrokePath += `M${x},${y}`;
                } else {
                  oneStrokePath += ` L${x},${y}`;
                }
              }
          }
        } else {
          for (let k = 0; k < strokes.x.length; k += 1) {
            const xpt = strokes.x[k];
            const ypt = strokes.y[k];
            if (k === 0) {
              oneStrokePath += `M${xpt},${ypt}`;
            } else {
              oneStrokePath += ` L${xpt},${ypt}`;
            }
          }
        }
        if (oneStrokePath !== undefined && oneStrokePath.length > 0) {
          let strokeClass = 'stdstroke';
          if (wrongStrokeDirectionAt === j + 1 || wrongStrokeAt === j + 1) {
            strokeClass = 'errstroke';
          }
          newSvgContent += `<path class="${strokeClass}" d="${oneStrokePath}"/>`;
          if (stdStrokeNum !== undefined) {
            newSvgContent += '<path class="stdstroke"/>';
          }
        }
      }
      hanSvg.innerHTML = `\n<path class="stdstroke"/>${newSvgContent}\n<path class="stdstroke"/>`;

      const svgPath = document.querySelectorAll('path.stdstroke, path.errstroke');

      anime({
        targets: svgPath,
        loop: true,
        direction: 'normal',
        strokeDashoffset: [anime.setDashoffset, 0],
        easing: 'easeInOutSine',
        duration: 1200,
        delay: (el, stroke) => (stroke * 1200),
      });
    },
    clearHanSvg() {
      const oldStrokes = document.querySelectorAll('path.stdstroke, path.errstroke');
      for (let i = oldStrokes.length - 1; i >= 0; i -= 1) {
        if (oldStrokes[i] && oldStrokes[i].parentElement) {
          oldStrokes[i].parentElement.removeChild(oldStrokes[i]);
        }
      }
    },
    useBezierCurve(xpts, ypts) {
      if (xpts.length !== ypts.length || xpts.length < 3 || xpts.length > 7) {
        return false;
      }
      const isXSimpleIncrease = xpts[0] < xpts[1];
      const isYSimpleIncrease = ypts[0] < ypts[1];
      for (let i = 2; i < xpts.length; i += 1) {
        if (isXSimpleIncrease !== (xpts[i - 1] < xpts[i])
            || isYSimpleIncrease !== (ypts[i - 1] < ypts[i])) {
          return false;
        }
      }
      return true;
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
@import url("myscript/dist/myscript.min.css");
h1,
h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
.writing-container {
  min-height: 600px;
  min-width: 300px;
}

#result {
  height: 49px;
  margin-left: 38px;
  margin-right: 38px;
  position: fixed;
  bottom: 0;
}

#message {
  display: none;
}

#errorMsg {
  height: 49px;
  position: fixed;
  bottom: 0;
  color: red;
  z-index: 10;
  background-color: #fffbe5;
  width: 100%;
  margin-left: 200px;
  text-align: left;
  padding: 5px 38px 5px 25px;
}

#hanSvg{
  position: fixed;
  bottom: 0;
  display: block;
  z-index: 20;
  background-color: #ffffff;
}

path.migrid,
path.migriddot {
  fill: none;
  stroke: #990000;
  stroke-width: 1px;
}

path.migriddot {
  stroke-dasharray: 4 2;
}

path.stdstroke,
path.errstroke {
  fill: none;
  stroke: #000000;
  stroke-width: 10px;
  stroke-linecap: round;
  stroke-linejoin: round;
}

path.errstroke {
  stroke: #c93756;
}
</style>
