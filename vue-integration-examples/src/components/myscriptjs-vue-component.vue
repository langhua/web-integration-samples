<template>
  <div>
    <h1>{{ msg }}</h1>
    <h3>{{ langhuaHandwritingMsg }}</h3>
    <div class="writing-container" touch-action="none" ref="editor" id="editor"></div>
    <div id="result" ref="result"></div>
    <div id="errorMsg" ref="errorMsg"></div>
  </div>
</template>

<script>
import * as MyScriptJS from 'myscript/src/myscript';

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
        if (this.idleEnd - this.idleStart < 2000) {
          window.setTimeout(() => {
            this.$refs.editor.editor.waitForIdle();
          }, (2100 + this.idleStart) - this.idleEnd);
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
            this.$refs.errorMsg.style.display = 'block';
          } else {
            this.$refs.errorMsg.innerHTML = '';
            this.$refs.errorMsg.style.display = 'none';
          }
          if (value.result === 'success' && value.score > 0) {
            this.$refs.result.innerHTML = this.$refs.result.innerHTML
              ? `${this.$refs.result.innerHTML}, ${value.han}:${value.message}/${value.score}`
              : `${value.han}:${value.message}/${value.score}`;
          }
        })
        .catch((error) => {
          window.console.log(error);
        });
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
  width: 100%;
  text-align: left;
}

#errorMsg {
  height: 49px;
  position: fixed;
  bottom: 0;
  color: red;
  z-index: 10;
  background-color: #fffbe5;
  width: 100%;
  display: none;
  padding: 5px 38px 5px 38px;
  text-align: left;
}
</style>
