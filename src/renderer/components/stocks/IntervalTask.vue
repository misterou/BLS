<template>
  <div class="IntervalTask">
      <el-collapse>
        <el-collapse-item name="每日签到" title="每日签到">
            <el-row :gutter="20">
              <el-col :span="16">
                  <el-tag type="warning">上次运行时间: {{DailySign.LastRun}}</el-tag>
              </el-col>
              <el-col :span="8">
                  <el-button type="success" size="small" @click="BtnClick_Sign">手动执行</el-button>
              </el-col>
            </el-row>
        </el-collapse-item>
        <el-collapse-item name="每日宝箱" title="每日宝箱">
            <el-row :gutter="20">
              <el-col :span="16">
                  <el-tag type="warning">上次运行时间: {{SilverBox.LastRun}}</el-tag>
              </el-col>
              <el-col :span="8">
                  <el-button type="success" size="small" @click="Silver">手动执行</el-button>
              </el-col>
            </el-row>
        </el-collapse-item>
        <el-collapse-item name="保持在线" title="保持在线">
            <el-row :gutter="20">
              <el-col :span="16">
                  <el-tag type="warning">上次运行时间: {{HeartBeat.LastRun}}</el-tag>
              </el-col>
            </el-row>
        </el-collapse-item>
      </el-collapse>
  </div>
</template>

<script>
import { CronJob } from "cron";
import { setTimeout, clearTimeout } from "timers";
export default {
  name: "IntervalTask",
  data() {
    return {
      DailySign: {
        LastRun: "无"
      },
      SilverBox: {
        LastRun: "无"
      },
      HeartBeat: {
        LastRun: "无"
      },
      dailyWishes: {
        open: false
      }
    };
  },
  mounted() {
    this.addListener(); //添加监听器
    this.GloBalTimeStampes(); //全局计时事件
  },
  methods: {
    addListener() {
      this.$eve.on("dailyTick", () => {
        this.Sign();
        this.DailyTask(); //顺便做一下每日任务
        this.TuanSign(); //应援团签到
        this.Silver(); //宝箱
      });
      this.$eve.on("HeartBeat", () => {
        this.KeepAlive();
      });
    },
    BtnClick_Sign(){
      //手动触发每日签到
      this.Sign();
      this.DailyTask(); //顺便做一下每日任务
      this.TuanSign(); //应援团签到
    },
    sleep(ms) {
      return new Promise(reject => {
        setTimeout(() => {
          reject(ms);
        }, ms);
      });
    },
    async HeartBeatSend(user) {
      let it = this.$api.use(user);
      it.headers["Referer"] = `https://live.bilibili.com/${
        this.$store.DanmakuRoom
      }`; //设置referer
      let pc = await it.send("User/userOnlineHeart", {}, "post");
      it.origin({
        /* APP端心跳 */
        uri: `https://api.live.bilibili.com/mobile/userOnlineHeart`,
        qs: user.SignWithBasicQuery({
          access_key: user.token.access_token
        }),
        form: {
          room_id: this.$store.DanmakuRoom,
          scale: "xxhdpi"
        },
        method: "post"
      });
    },
    KeepAlive() {
      this.HeartBeat.LastRun = this.formatTime(); //更新最后执行时间

      for (let user of this.$store.users) {
        if (user.config.KeepAlive && user.isLogin) {
          this.HeartBeatSend(user);
        }
      }
    },
    async getSilver(user) {
      try {
        let rq = await this.$api.use(user).send(
          "mobile/freeSilverCurrentTask",
          user.SignWithBasicQuery({
            access_key: user.token.access_token
          })
        );
        if (rq.code == 0) {
          let time = rq.data.minute * 6e4;
          await this.sleep(time);
          let sl = await this.$api.use(user).send(
            "mobile/freeSilverAward",
            user.SignWithBasicQuery({
              access_key: user.token.access_token
            })
          );
          this.getSilver(user); // 递归
        } else if (rq.code == -10017) {
          this.$eve.emit("info", `${user.name} 领宝箱: ${rq.msg}`);
          /* 今日宝箱已领完 */
        }
      } catch (e) {
        this.$eve.emit("info", `${user.name} 领取银瓜子宝箱 : ${e.message}`);
      }
    },
    Silver() {
      this.SilverBox.LastRun = this.formatTime(); //更新最后运行时间

      for (let user of this.$store.users) {
        if (user.config.SilverBox && user.isLogin) {
          this.getSilver(user);
        }
      }
    },
    DailyTask() {
      for (let user of this.$store.users) {
        if (user.isLogin) {
          this.$api
            .use(user)
            .send(
              "activity/v1/task/receive_award",
              { task_id: "double_watch_task" },
              "post"
            );
        }
      }
    },
    async userTuanSign(user){
        try{
            let list  = await this.$api.use(user).origin(
              {
                uri:"https://api.vc.bilibili.com/link_group/v1/member/my_groups",
                qs:user.SignWithBasicQuery({
                    access_key: user.token.access_token
                  }),
                method:"get",
            });
            if(list.code===0){
                for(let group of list.data.list){
                    let sign = await this.$api.use(user).origin(
                      {
                        uri:"https://api.vc.bilibili.com/link_setting/v1/link_setting/sign_in",
                        qs:user.SignWithBasicQuery({
                            group_id:group.group_id,
                            owner_id:group.owner_uid,
                            access_key: user.token.access_token,
                        }),
                        method:"get",
                    });
                    this.$eve.emit("info",`${user.name} 应援团 [ ${group.group_name} ] 签到 :${sign.msg}`);
                    //发送请求即可
                }
            }
        }catch(e){
            this.$eve.emit("info",`${user.name} 应援团签到时: ${e.message}`);
        }
    },
    TuanSign() {
        for(let user of this.$store.users){
            if(user.config.DailySign && user.isLogin){
                this.userTuanSign(user);
            }
        }
    },
    async Sign() {
      this.DailySign.LastRun = this.formatTime(); //更新最后运行时间

      for (let user of this.$store.users) {
        if (user.config.DailySign && user.isLogin) {
          try {
            this.$api
              .use(user)
              .send("gift/v2/live/receive_daily_bag", {}, "get"); //每日包裹，不重要
            let rq = await this.$api.use(user).send("sign/doSign", {}, "get"); //签到
            this.$eve.emit("info", `${user.name} 签到: ${rq.msg}`);
          } catch (e) {
            this.$eve.emit("info", `${user.name} 签到: ${e.message}`);
          }
        }
      }
    },
    GloBalTimeStampes() {
      /* 负责在初始化之后提供全局计时事件 */
      setTimeout(() => {
        this.$eve.emit("dailyTick");
      }, 10e3); //载入10秒后提交一个dailyTick
      /* 使用cron进行定时任务,每天凌晨1点emit一个dailyTick事件 */
      let dailyjob = new CronJob(
        "00 00 01 * * *",
        () => {
          this.$eve.emit("dailyTick"); // 触发dailyTick事件
        },
        null,
        true
        // timeZone:"China/Shanghai",
      );
      this.$eve.emit("HeartBeat"); // 触发心跳请求
      let heartjob = new CronJob(
        "0 */5 * * * *",
        () => {
          this.$eve.emit("HeartBeat"); // 触发心跳请求
        },
        null,
        true
        // timeZone:"China/Shanghai",
      );
    },
    formatTime(date = new Date(), fmt = "YYYY-MM-DD HH:mm:ss") {
      date = typeof date === "number" ? new Date(date) : date;
      var o = {
        "M+": date.getMonth() + 1,
        "D+": date.getDate(),
        "h+": date.getHours() % 12 === 0 ? 12 : date.getHours() % 12,
        "H+": date.getHours(),
        "m+": date.getMinutes(),
        "s+": date.getSeconds(),
        "q+": Math.floor((date.getMonth() + 3) / 3),
        S: date.getMilliseconds()
      };
      var week = {
        "0": "\u65e5",
        "1": "\u4e00",
        "2": "\u4e8c",
        "3": "\u4e09",
        "4": "\u56db",
        "5": "\u4e94",
        "6": "\u516d"
      };
      if (/(Y+)/.test(fmt)) {
        fmt = fmt.replace(
          RegExp.$1,
          (date.getFullYear() + "").substr(4 - RegExp.$1.length)
        );
      }
      if (/(E+)/.test(fmt)) {
        fmt = fmt.replace(
          RegExp.$1,
          (RegExp.$1.length > 1
            ? RegExp.$1.length > 2 ? "\u661f\u671f" : "\u5468"
            : "") + week[date.getDay() + ""]
        );
      }
      for (var k in o) {
        if (new RegExp("(" + k + ")").test(fmt)) {
          fmt = fmt.replace(
            RegExp.$1,
            RegExp.$1.length === 1
              ? o[k]
              : ("00" + o[k]).substr(("" + o[k]).length)
          );
        }
      }
      return fmt;
    }
  }
};
</script>
<style scoped>

</style>
