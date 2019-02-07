<template>
<div class="game">
    <Header :back="true" :size="15"/>
    <Map/>
    <Footer/>
</div>
</template>

<script>
// @ is an alias to /src
import Header from '@/components/Header.vue'
import Footer from '@/components/Footer.vue'
import Map from '@/components/Map.vue'

export default {
    name: 'game',
    components: {
    Header,Footer,Map
    }
}
</script>
