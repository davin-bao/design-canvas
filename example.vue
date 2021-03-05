<template>
	<view class="content">
        <view class="content">
            <m-canvas ref="canvas" :backgroundUrl="img_url" />
        </view>
	</view>
</template>

<script>
	import mCanvas from '@/components/m-canvas/m-canvas'
	import { variationShow } from '@/api/listing'

	export default {
		components: {
			mCanvas
		},
		data() {
			return {
				img_url: '',		// 背景图片
				renderCanvas: false,
				preview_image: '',	// 预览图
				printed_image: [],  //自定义设计元素
			};
		},
		mounted() {
			this.variationId = this.listing_variation_id
			this.loadVariation()
		},
		methods: {
			onReDraw: function(e){
				const canvas = this.$refs.canvas
				canvas.draw()
			},
			complated: function(callback){
				this.saveData(callback)
			},
			saveData: function(callback){
				const self = this
				setTimeout(()=>{
					const item = this.$refs.canvas
					item.save((previewImage, printedImages)=>{
						if(!previewImage && !printedImages){
							callback(false)
							return
						}
						self.preview_image = previewImage
						self.printed_image = printedImages
					})
				}, 200)
				
			}
		}
	};
</script>

<style>
	
</style>
