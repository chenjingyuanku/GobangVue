<template>
    <div>
        <div class="button" @click="goGame(0)">
            人人模式
        </div>
        <div class="button" @click="goGame(1)">
            人机机先
        </div>
        <div class="button" @click="goGame(2)">
            人机人先
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
@Component
export default class Menu extends Vue{
    goGame(mode:number){
        let pageInfo:object = {
            name:"game",
            params:{
                mode:mode
            }
        }
        this.$router.push(pageInfo)
    }
}
</script>

<style scoped>
.button{
    user-select: none;
    cursor: pointer;
    background-color: rgb(194,149,107);
    margin: auto;
    margin-top: 24px;
    text-align: center;
    width: 60%;
    height: 50px;
    line-height: 50px;
    box-shadow: 4px 4px 4px gray;
    border-radius: 10px;
}
.button:active{
    box-shadow: 1px 1px 1px gray;
    transform: translateY(4px);
}
/*手机等设备风格*/
@media screen and (max-width: 600px) {
    .button{
        width: 100%;
        font-weight: bold;
        justify-content: center;
    }
}
</style>

