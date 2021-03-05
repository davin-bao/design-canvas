<template>
	<view>
		<canvas :id="'uni-canvas'+identifier" :canvas-id="'uni-canvas'+identifier" class="main-canvas" :style="{height: cvs.height}"
		 disable-scroll="false" @touchmove='onMove' @touchstart='onStart($event)' @touchend='onEnd' @touchcancel='onCancel'
		 @longtap='onTap' @error='onError'></canvas>
		<view class="toolbar">
			<cmd-bottom-nav :list="actionList" @click="onBottomNavClick"></cmd-bottom-nav>
		</view>
		<picker-color :isShow="pickColorShow" :bottom="0" @callback='onTxtColorEdit'></picker-color>
		<view class="cu-modal" :class="txtEditShow ? 'show' : ''">
			<view class="cu-dialog">
				<view class="cu-bar bg-white justify-end">
					<view class="content">{{$t('components.mCanvas.form.insertTxt')}}</view>
					<view class="action" @tap="onClose">
						<text class="cuIcon-close text-red"></text>
					</view>
				</view>
				<view>
					<input class="uni-input text-lg no-padding" v-model="insertTxt.content" :placeholder="$t('components.mCanvas.tips.insertTxt')" />
				</view>
				<view class="cu-bar bg-white">
					<view class="action margin-0 flex-sub text-grey solid-left" @tap="onClose">{{$t('components.button.cancel')}}</view>
					<view class="action margin-0 flex-sub text-blue solid-left" @tap="onTxtEdit">{{$t('components.button.ok')}}</view>
				</view>
			</view>
		</view>
		<wzpPicker ref="wzpPicker" mode="one" :pickerList="txtStyle" pickerKey="label" :defaultIndex="insertTxt.style" @onConfirm="onTxtStyleEdit" />
		<cameraModal v-if="cameraModal" @ChooseImage="insertImage"></cameraModal>
		<view class="cu-load load-cuIcon loading" v-if="loading"></view>
	</view>
</template>
<script>
	import cmdBottomNav from "./cmd-bottom-nav/cmd-bottom-nav"
	import wzpPicker from './wzp-picker/wzpPicker'
	import pickerColor from "./helang-pickerColor/helang-pickerColor"
	import cameraModal from './cameraModal/cameraModal.vue';

	var touchs = [];
	var canvasw = 0;
	var canvash = 0;
	var selectError = 25; //选择器误差
	var selectStrokeColor = '#AAAAAA';

	//获取系统信息
	uni.getSystemInfo({
		success: function(res) {
			canvasw = res.windowWidth;
			canvash = res.windowWidth * 440 / 375;
		},
	})

	export default {
		name: "m-canvas",
		props: {
			identifier: {
				default: 0
			},
			backgroundUrl: {
				type: String,
				required: true
			},
			data: {
				type: Array
			}
		},
		components: {
			cmdBottomNav,
			"picker-color": pickerColor,
			wzpPicker,
			cameraModal
		},
		data() {
			return {
				loading: true,
				context: null,
				cvs: {
					height: 0
				},
				backgroud: null,
				layers: [],
				selected: -1,
				mode: '',
				txtStyle: [{
						value: 0,
						label: 'Arial'
					},
					{
						value: 1,
						label: 'sans-serif'
					},
					{
						value: 2,
						label: 'serif'
					},
					{
						value: 3,
						label: 'monospace'
					},
					{
						value: 4,
						label: 'cursive'
					},
					{
						value: 5,
						label: 'fantasy'
					}
				],
				pickTxtStyle: `height: ${Math.round(uni.getSystemInfoSync().screenHeight/2)}px;`,
				txtEditShow: false,
				pickColorShow: false,
				insertTxt: {
					content: '',
					style: [0],
					color: '#AAAAAA'
				},
				actionList: [],
				//保存的图片路径
				previewImage: '',
				printedImages: [],
				cameraModal: false,
			}
		},
		watch: {
			data(val) {
				this.layers = val
			},
			layers(val) {
				this.draw()
			}
		},
		mounted: function() {
			this.context = uni.createCanvasContext('uni-canvas' + this.identifier, this)
			this.cvs.height = canvash + 'px'
			this.bottomNavChange()
			this.drawBackground()
		},
		methods: {
			save: function(callback) {
				const self = this
				this.printedImages = []
				//保存当前预览图
				const canvasIdentifier = 'uni-canvas' + this.identifier
				uni.canvasToTempFilePath({
					canvasId: canvasIdentifier,
					quality: 0.8,
					success: function(res) {
						self.previewImage = res.tempFilePath
						self.saveLayer(0, callback)
					},
					fail: function(err) {
						console.log('canvas to temp file fail', err)
						uni.showToast({
							title: $t('compontents.mCanvas.tips.saveImgFailed'),
							duration: 2000
						})
						setTimeout(() => {
							callback(false, false)
						}, 2000)
					}
				}, this)

			},
			saveLayer: function(index, callback) {
				const self = this
				if (index >= this.layers.length) {
					callback(this.previewImage, this.printedImages)
					return
				}

				const item = this.layers[index]
				const context = this.context
				context.clearRect(0, 0, canvasw, canvash)

				context.translate(item.pos.x, item.pos.y)
				context.rotate(item.rot)
				switch (item.type) {
					case 'img':
						context.drawImage(item.url, -item.size.w / 2, -item.size.h / 2, item.size.w, item.size.h)
						break
					case 'txt':
						context.setFillStyle(item.color)
						context.font = item.size.h + 'px ' + this.txtStyle[item.style].label
						context.fillText(item.content, -item.size.w / 2, -item.size.h / 2 + item.size.h * 9 / 10)
						break
				}
				const box = this.calculateBox(item.size, item.rot);

				context.draw(true, () => {
					uni.canvasToTempFilePath({
						// x: item.pos.x-box.w/2-1,
						// y: item.pos.y-box.h/2-1,
						// width: box.w,
						// height: box.h,
						// destWidth: box.w+2,
						// destHeight: box.h+2, 
						canvasId: 'uni-canvas' + self.identifier,
						quality: 0.8,
						success: function(res) {
							self.printedImages.push(res.tempFilePath)
							self.saveLayer(index + 1, callback)
						},
						fail: function(err) {
							console.log('canvas to temp file fail', err)
							uni.showToast({
								title: $t('compontents.mCanvas.tips.saveImgFailed'),
								duration: 2000
							})
							setTimeout(() => {
								callback(false, false)
							}, 2000)
						}
					}, self)
				})
			},
			/* UI操作 */
			bottomNavChange: function() {
				const item = this.layers[this.selected]

				this.insertTxt = {
					content: '',
					color: '#AAAAAA',
					style: [0],
				}

				if (!item) {
					this.actionList = [{
						"text": this.$t('components.mCanvas.actions.image'),
						"icon": "image"
					}, {
						"text": this.$t('components.mCanvas.actions.txt'),
						"icon": "font"
					}]
				} else if (item.type == 'txt') {
					this.actionList = [{
						"text": this.$t('components.mCanvas.actions.edit'),
						"icon": "edit"
					}, {
						"text": this.$t('components.mCanvas.actions.style'),
						"icon": "text-height"
					}, {
						"text": this.$t('components.mCanvas.actions.color'),
						"icon": "braille"
					}, {
						"text": this.$t('components.mCanvas.actions.trash'),
						"icon": "trash"
					}]

					this.insertTxt.content = item.content
					this.insertTxt.color = item.color
					this.insertTxt.style = [item.style]
				}
			},
			onBottomNavClick: function(e) {
				switch (e.item.icon) {
					case 'image':
						this.cameraModal = true;
						break
					case 'font':
						this.txtEditShow = true
						break
					case 'edit':
						this.txtEditShow = true
						break
					case 'text-height':
						this.$refs.wzpPicker.showPicker()
						break
					case 'braille':
						this.pickColorShow = true
						break
					case 'trash':
						this.layers.splice(this.selected, 1)
						this.selected = -1
						break
				}
			},
			insertImage: function(params) {
				if (params.flag) {
					let self = this
					if (this.fSelecting) return;
					this.fSelecting = true;
					setTimeout(() => {
						this.fSelecting = false;
					}, 500);
					uni.chooseImage({
						count: 1,
						sizeType: ['original', 'compressed'],
						sourceType: [params.index ? 'album' : 'camera'],
						success: (r) => {
							uni.showLoading({
								mask: true
							});
							let path = this.imgPath = r.tempFilePaths[0];
							uni.getImageInfo({
								src: path,
								success: r => {
									this.layers.push(this.fixObj({
										type: 'img',
										url: path,
										size: {
											w: r.width,
											h: r.height
										}
									}))
								},
								fail: () => {
									uni.showToast({
										title: $t('compontents.mCanvas.tips.getImgFailed'),
										duration: 2000,
									})
								},
								complete() {
									uni.hideLoading();
								}
							});
							this.cameraModal = false;
						}
					})
				} else {
					this.cameraModal = false;
				}
			},
			onTxtEdit: function() {
				this.txtEditShow = false

				let item = this.layers[this.selected]
				if (item && item.type === 'txt') {
					this.context.font = this.layers[this.selected].size.h + 'px ' + this.txtStyle[this.insertTxt.style[0]].label
					const measure = this.context.measureText(this.insertTxt.content)

					this.layers[this.selected].content = this.insertTxt.content
					this.layers[this.selected].size.w = measure && measure.width || 100
				} else {
					this.context.font = '30px ' + this.txtStyle[this.insertTxt.style[0]].label
					const measure = this.context.measureText(this.insertTxt.content)

					const newItem = this.fixObj({
						type: 'txt',
						size: {
							w: measure && measure.width || 100,
							h: 30
						},
						content: this.insertTxt.content,
						color: this.insertTxt.color,
						style: this.insertTxt.style[0]
					})
					this.layers.push(newItem)
				}
				this.draw()
			},
			onClose: function() {
				this.txtEditShow = false
			},
			onTxtStyleEdit: function(e) {
				if (e && e.length > 0) {
					this.insertTxt.style = e
					let item = this.layers[this.selected]
					if (item) {
						this.context.font = this.layers[this.selected].size.h + 'px ' + this.txtStyle[this.insertTxt.style[0]].label
						const measure = this.context.measureText(this.insertTxt.content)

						this.layers[this.selected].size.w = measure && measure.width || 100
						this.layers[this.selected].style = this.insertTxt.style[0]
					}
					this.draw()
				}
			},
			onTxtColorEdit: function(color) {
				this.pickColorShow = false;
				if (color) {
					this.insertTxt.color = color

					let item = this.layers[this.selected]
					if (item) {
						this.layers[this.selected].color = color
					}
					this.draw()
				}
			},
			/* UI操作结束 */
			/* 画布操作 */
			fixObj: function(obj) {
				obj.rot = 0
				if (obj.size) {
					if (obj.size.w > canvasw || obj.size.h > canvash) {
						if (obj.size.w / canvasw > obj.size.h / canvash) {
							obj.size.h = obj.size.h * canvasw / obj.size.w / 2
							obj.size.w = canvasw / 2
						} else {
							obj.size.w = obj.size.w * canvash / obj.size.h / 2
							obj.size.h = canvash / 2
						}
					}

					if (!obj.pos) {
						obj.pos = {
							x: canvasw / 2,
							y: canvash / 2,
						}
					}
				}
				return obj
			},
			onStart: function(e) {
				//获取触摸开始的 x,y
				let point = {
					x: e.changedTouches[0].x,
					y: e.changedTouches[0].y
				}
				this.judgeMode(point)
			},
			// 画布的触摸移动手势响应
			onMove: function(e) {
				let point = {
					x: e.touches[0].x,
					y: e.touches[0].y
				}

				touchs.push(point)
				if (touchs.length >= 2 && this.layers[this.selected]) {
					const item = this.layers[this.selected]
					if (!item) return;
					switch (this.mode) {
						case 'move':
							const thur = Math.PI / 2 - item.rot - Math.atan(item.size.h / item.size.w)
							const c = Math.sqrt(Math.pow(item.size.w, 2) + Math.pow(item.size.h, 2)) / 2
							item.pos.x = point.x + c * Math.sin(thur)
							item.pos.y = point.y + c * Math.cos(thur)

							break
						case 'rotate':
							item.rot = this.calculateAngle({
								x: item.pos.x,
								y: item.pos.y,
							}, point) - this.calculateAngle({
								x: item.pos.x,
								y: item.pos.y
							}, {
								x: item.pos.x + item.size.w / 2,
								y: item.pos.y - item.size.h / 2
							})
							break
						case 'scale':
							const thur1 = Math.atan(item.size.h / item.size.w)
							const c1 = Math.sqrt(Math.pow((point.x - item.pos.x), 2) + Math.pow((point.y - item.pos.y), 2))

							let w = c1 * Math.sin(thur1)
							let h = c1 * Math.cos(thur1)
							w = (w > canvasw - 20) ? canvasw - 20 : w
							w = (w < 20) ? 20 : w
							h = (h > canvash - 20) ? canvash - 20 : h
							h = (h < 20) ? 20 : h

							if (w > h * item.size.w / item.size.h) h = w * item.size.h / item.size.w
							else w = h * item.size.w / item.size.h

							item.size.w = Math.round(w)
							item.size.h = Math.round(h)
							break
						case 'delete':
							//在 judgeMode 中处理
							break
					}
					this.draw()
				}
			},
			// 画布的触摸移动结束手势响应
			onEnd: function(e) {
				//清空轨迹数组
				for (let i = 0; i < touchs.length; i++) {
					touchs.pop()
				}
			},
			// 画布的触摸取消响应
			onCancel: function(e) {},
			// 画布的长按手势响应
			onTap: function(e) {},
			onError: function(e) {},
			draw: function() {
				const context = this.context
				context.clearRect(0, 0, canvasw, canvash)
				//绘制背景图
				this.drawBackground()

				this.layers.map((item, index) => {
					context.translate(item.pos.x, item.pos.y)
					context.rotate(item.rot)
					switch (item.type) {
						case 'img':
							context.drawImage(item.url, -item.size.w / 2, -item.size.h / 2, item.size.w, item.size.h)
							break
						case 'txt':
							context.setFillStyle(item.color)
							context.font = item.size.h + 'px ' + this.txtStyle[item.style].label

							context.fillText(item.content, -item.size.w / 2, -item.size.h / 2 + item.size.h * 9 / 10)
							break
					}

					if (index === this.selected) {
						context.setStrokeStyle(selectStrokeColor)
						context.setLineWidth(1)
						//选择框
						context.translate(-item.size.w / 2, -item.size.h / 2)
						context.beginPath()
						context.moveTo(selectError / 2, 0);
						context.lineTo(-selectError / 2 + item.size.w, 0);
						context.moveTo(selectError / 2, item.size.h);
						context.lineTo(-selectError / 2 + item.size.w, item.size.h);
						context.moveTo(0, selectError / 2);
						context.lineTo(0, -selectError / 2 + item.size.h);
						context.moveTo(0 + item.size.w, selectError / 2);
						context.lineTo(0 + item.size.w, -selectError / 2 + item.size.h);
						context.stroke()
						context.translate(item.size.w / 2, item.size.h / 2)

						//移动句柄
						context.translate(-item.size.w / 2, -item.size.h / 2)
						context.strokeRect(
							-selectError / 2,
							-selectError / 2,
							selectError,
							selectError
						)
						context.beginPath()
						context.moveTo(-selectError / 4, 0);
						context.lineTo(selectError / 4, 0);
						context.moveTo(0, -selectError / 4);
						context.lineTo(0, selectError / 4);
						//左箭头
						context.moveTo(-selectError / 4, -selectError / 8);
						context.lineTo(-selectError * 3 / 8, 0);
						context.lineTo(-selectError / 4, selectError / 8);
						//右箭头
						context.moveTo(selectError / 4, -selectError / 8);
						context.lineTo(selectError * 3 / 8, 0);
						context.lineTo(selectError / 4, selectError / 8);
						//上箭头
						context.moveTo(-selectError / 8, -selectError / 4);
						context.lineTo(0, -selectError * 3 / 8);
						context.lineTo(selectError / 8, -selectError / 4);
						//下箭头
						context.moveTo(-selectError / 8, selectError / 4);
						context.lineTo(0, selectError * 3 / 8);
						context.lineTo(selectError / 8, selectError / 4);
						context.stroke()
						context.translate(item.size.w / 2, item.size.h / 2)

						//旋转句柄
						context.translate(item.size.w / 2, -item.size.h / 2)
						context.strokeRect(
							-selectError / 2,
							-selectError / 2,
							selectError,
							selectError
						)
						context.beginPath()
						context.arc(0, 0, selectError / 4, 0, Math.PI / 4, true)
						context.stroke()
						//箭头
						context.beginPath()
						context.setLineWidth(2)
						context.moveTo(selectError / 8, -selectError / 8);
						context.lineTo(selectError / 4, 0);
						context.lineTo(selectError / 4 + selectError / 16, -selectError / 6);
						context.stroke()
						context.translate(-item.size.w / 2, item.size.h / 2)

						//缩放句柄
						context.translate(item.size.w / 2, item.size.h / 2)
						context.setLineWidth(1)
						context.strokeRect(
							-selectError / 2,
							-selectError / 2,
							selectError,
							selectError
						)
						//斜线
						context.beginPath()
						context.moveTo(-selectError / 4, -selectError / 4);
						context.lineTo(selectError / 4, selectError / 4);
						//左上箭头
						context.moveTo(-selectError * 3 / 8, -selectError / 8);
						context.lineTo(-selectError * 3 / 8, -selectError * 3 / 8);
						context.lineTo(-selectError * 1 / 8, -selectError * 3 / 8);
						//右下箭头
						context.moveTo(selectError * 3 / 8, selectError / 8);
						context.lineTo(selectError * 3 / 8, selectError * 3 / 8);
						context.lineTo(selectError * 1 / 8, selectError * 3 / 8);
						context.stroke()
						context.translate(-item.size.w / 2, -item.size.h / 2)

						//删除句柄
						context.translate(-item.size.w / 2, item.size.h / 2)
						context.setLineWidth(1)
						context.strokeRect(
							-selectError / 2,
							-selectError / 2,
							selectError,
							selectError
						)
						//斜线一
						context.beginPath()
						context.moveTo(-selectError / 4, -selectError / 4);
						context.lineTo(selectError / 4, +selectError / 4);
						//斜线二
						context.moveTo(selectError / 4, -selectError / 4);
						context.lineTo(-selectError / 4, selectError / 4);
						context.stroke()
						context.translate(item.size.w / 2, -item.size.h / 2)
					}

					context.rotate(-item.rot)
					context.translate(-item.pos.x, -item.pos.y)
				})

				context.draw()
			},
			drawBackground: function() {
				const self = this
				if (!self.backgroud) {

					uni.getImageInfo({
						src: self.backgroundUrl,
						success(res) {
							self.backgroud = self.fixObj({
								url: res.path,
								size: {
									w: res.width,
									h: res.height
								}
							})
							self.backgroud.size.w *= 2
							self.backgroud.size.h *= 2
							self.context.translate(self.backgroud.pos.x, self.backgroud.pos.y)

							self.context.drawImage(self.backgroud.url, -self.backgroud.size.w / 2, -self.backgroud.size.h / 2, self.backgroud
								.size.w, self.backgroud.size.h)
							self.context.translate(-self.backgroud.pos.x, -self.backgroud.pos.y)
							self.context.draw()

							setTimeout(() => {
								self.loading = false
							}, 3000)
						},
						fail(error) {
							console.log(error)
							uni.showToast({
								title: $t('compontents.mCanvas.tips.getImgFailed'),
								duration: 2000,
							})
						}
					})
				} else {
					self.context.translate(self.backgroud.pos.x, self.backgroud.pos.y)
					self.context.drawImage(self.backgroud.url, -self.backgroud.size.w / 2, -self.backgroud.size.h / 2, self.backgroud
						.size.w, self.backgroud.size.h)
					self.context.translate(-self.backgroud.pos.x, -self.backgroud.pos.y)
				}
			},
			select: function(point) {
				let selected = -1
				this.layers.map((item, index) => {
					const newPoint = this.calculatePoint({
						x: point.x - item.pos.x,
						y: point.y - item.pos.y
					}, item.rot)
					const selectRect = [{
							x: -item.size.w / 2 - selectError / 2,
							y: -item.size.h / 2 - selectError / 2
						},
						{
							x: -item.size.w / 2 - selectError / 2,
							y: item.size.h / 2 + selectError / 2
						},
						{
							x: item.size.w / 2 + selectError / 2,
							y: -item.size.h / 2 - selectError / 2
						},
						{
							x: item.size.w / 2 + selectError / 2,
							y: -item.size.h / 2 + selectError / 2
						}
					]

					if (this.inBox(newPoint, selectRect)) {
						selected = index
					}
				})

				return selected
			},
			judgeMode: function(point) {
				this.mode = ''
				let selected = this.select(point)
				if (this.selected !== selected) {
					// 选择新的对象
					this.selected = selected
				} else {
					const item = this.layers[this.selected]
					if (item) {
						//原点
						let x = item.pos.x
						let y = item.pos.y
						const newPoint = this.calculatePoint({
							x: point.x - item.pos.x,
							y: point.y - item.pos.y
						}, item.rot)

						const moveRect = [{
								x: -item.size.w / 2 - selectError / 2,
								y: -item.size.h / 2 - selectError / 2
							},
							{
								x: -item.size.w / 2 + selectError / 2,
								y: -item.size.h / 2 - selectError / 2
							},
							{
								x: -item.size.w / 2 + selectError / 2,
								y: -item.size.h / 2 + selectError / 2
							},
							{
								x: -item.size.w / 2 - selectError / 2,
								y: -item.size.h / 2 + selectError / 2
							}
						];

						if (this.inBox(newPoint, moveRect)) {
							this.mode = 'move'
						}

						const rotateRect = [{
								x: item.size.w / 2 - selectError / 2,
								y: -item.size.h / 2 - selectError / 2
							},
							{
								x: item.size.w / 2 + selectError / 2,
								y: -item.size.h / 2 - selectError / 2
							},
							{
								x: item.size.w / 2 + selectError / 2,
								y: -item.size.h / 2 + selectError / 2
							},
							{
								x: item.size.w / 2 - selectError / 2,
								y: -item.size.h / 2 + selectError / 2
							}
						];

						if (this.inBox(newPoint, rotateRect)) {
							this.mode = 'rotate'
						}

						const scaleRect = [{
								x: item.size.w / 2 - selectError / 2,
								y: item.size.h / 2 - selectError / 2
							},
							{
								x: item.size.w / 2 + selectError / 2,
								y: item.size.h / 2 - selectError / 2
							},
							{
								x: item.size.w / 2 + selectError / 2,
								y: item.size.h / 2 + selectError / 2
							},
							{
								x: item.size.w / 2 - selectError / 2,
								y: item.size.h / 2 + selectError / 2
							}
						];

						if (this.inBox(newPoint, scaleRect)) {
							this.mode = 'scale'
						}

						const deleteRect = [{
								x: -item.size.w / 2 - selectError / 2,
								y: item.size.h / 2 - selectError / 2
							},
							{
								x: -item.size.w / 2 + selectError / 2,
								y: item.size.h / 2 - selectError / 2
							},
							{
								x: -item.size.w / 2 + selectError / 2,
								y: item.size.h / 2 + selectError / 2
							},
							{
								x: -item.size.w / 2 - selectError / 2,
								y: item.size.h / 2 + selectError / 2
							}
						];
						if (this.inBox(newPoint, deleteRect)) {
							this.mode = 'delete'
							this.layers.splice(this.selected, 1)
							this.selected = -1
						}
					}
				}
				// 工具条变更
				this.bottomNavChange()
				this.draw()
			},
			/* 画布操作结束 */
			/* 辅助函数 */
			//根据直线1与直线2的坐标计算旋转角度
			calculateAngle: function(p1, p2) {
				let xDis
				let yDis
				xDis = p2.x - p1.x;
				yDis = p2.y - p1.y;
				let angle = Math.atan2(yDis, xDis)

				return angle;
			},
			//矩形旋转后的当前点所对应的世界坐标系的点
			calculatePoint: function(point, angle) {
				const sin = Math.sin(-angle)
				const cos = Math.cos(-angle)
				const newPos = {
					x: point.x * cos - point.y * sin,
					y: point.y * cos + point.x * sin
				}

				return newPos
			},
			//矩形旋转后的所对应的世界坐标系的大小
			calculateBox: function(size, angle) {
				return {
					w: Math.abs(size.h * Math.sin(angle)) + Math.abs(size.w * Math.cos(angle)),
					h: Math.abs(size.w * Math.sin(angle)) + Math.abs(size.h * Math.cos(angle))
				}
			},
			inBox: function(point, rect) {
				let x1 = 999,
					x2 = -999,
					y1 = 999,
					y2 = -999
				rect.map(item => {
					if (x1 > item.x) x1 = item.x
					if (x2 < item.x) x2 = item.x
					if (y1 > item.y) y1 = item.y
					if (y2 < item.y) y2 = item.y
				})

				if (
					point.x >= x1 &&
					point.x <= x2 &&
					point.y >= y1 &&
					point.y <= y2
				) return true

				return false
			},
			/* 辅助函数结束 */
		}
	}
</script>
<style>
	.main-canvas {
		display: flex;
		position: fixed !important;
		background: #efeff4;
		left: 0;
		width: 100%;
	}

	.toolbar {
		padding: 5upx;
		display: flex;
		position: fixed !important;
		bottom: 0;
	}

	.txt-modal {
		z-index: auto !important;
	}

	.cu-load {
		position: absolute;
		top: 50%;
		left: 50%;
	}

	.no-padding {
		padding: 0 !important;
	}
</style>
