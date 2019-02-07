<template>
    <div class="header">
        <img class="back-left" @click="goHome" v-if="back" src="../assets/back-left.png">
        <img class="logo" src="../assets/logo.png">
        <h1>五子棋</h1>
        <img class="back-right" @click="goHome" v-if="back" src="../assets/back-right.png">
    </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

@Component
export default class Header extends Vue{
    @Prop({default:false}) private back?: boolean;
    goHome(){
        this.$router.push({name:"home"});
    }
}
</script>

<style scoped>
/*电脑或平板模式下风格*/ 
.header{
    position:relative;
    user-select: none;
    cursor: pointer;
    font-family: "YouYuan","幼圆";
    width: 60%;
    min-width: 80vh;
    height: 36px;
    line-height: 36px;
    font-size: 10px;
    background-color: rgba(228, 228, 228, 0.5);
    margin: auto;
    box-shadow: 5px 5px 5px #888888;
    display: flex;
    align-items: center;
    /*border-radius: 10px;*/
}
.logo{
    height: 20px;
    line-height: inherit;
    margin: 0 5px 0 10px;
}
.back-left{
    display: none;
}
.back-right{
    height: 20px;
    clear: both;
    display: block;
    margin: auto;
    margin-right: 8px;
}
/*手机等设备风格*/
@media screen and (max-width: 600px) {
    .header{
        font-size: 14px;
        height: 40px;
        line-height: 40px;
        width: 100%;
        min-width: 100%;
        justify-content: center;
    }
    .logo{
        height: 28px;
        margin: 0;
    }
    .back-left{
        position: absolute;
        height: 20px;
        width: 20px;
        left: 10px;
        display: block;
    }
    .back-right{
        display: none;
    }
}
</style>

