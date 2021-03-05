<template>
	<view>
		<view class="picker-mask" @click="closePicker" catchtouchmove="true" v-show="pickerShow"></view>
		<view class="picker-content" :class="{'pickerShow':pickerShow}">
			<view class="picker-button">
				<text class="action text-grey" @click="closePicker">{{$t('components.button.cancel')}}</text>
				<text class="action text-blue" @click="confirm">{{$t('components.button.ok')}}</text>
			</view>
			<!-- 单列选择 -->
			<picker-view class="picker-view" indicator-style="height:100upx;" indicator-class="picker-view-selected-one" :value="pickerIndex" @change="pickerViewChangeOne"
			 v-if="mode=='one'">
				<picker-view-column>
					<view class="item picker-item" v-for="(item,index) in pickerList" :key="index">{{item[pickerKey]}}</view>
					<!-- <view class="" v-for="(item, index) in pickerList" :key="index">{{item.label}}</view> -->
				</picker-view-column>
			</picker-view>
			<!-- 双列选择-联动 -->
			<picker-view class="picker-view" indicator-style="height:100upx;" indicator-class="picker-view-selected-two" :value="pickerIndex" @change="pickerViewChangeTwo"
			 v-if="mode=='two_linkage'">
				<picker-view-column>
					<view class="picker-item" v-for="(item, index) in pickerList" :key="index">{{item[pickerKey]}}</view>
				</picker-view-column>
				<picker-view-column v-if="pickerList[pickerIndex[0]]">
					<view class="picker-item" v-for="(item, index) in pickerList[pickerIndex[0]].children" :key="index">{{item[pickerKey]}}</view>
				</picker-view-column>
			</picker-view>
			<!-- 双列选择-不联动 -->
			<picker-view class="picker-view" indicator-style="height:100upx;" indicator-class="picker-view-selected-two" :value="pickerIndex" @change="pickerViewChangeTwo"
			 v-if="mode=='two_nolinkage'">
				<picker-view-column>
					<view class="picker-item" v-for="(item, index) in pickerList.firstArray" :key="index">{{item[pickerKey]}}</view>
				</picker-view-column>
				<picker-view-column>
					<view class="picker-item" v-for="(item, index) in pickerList.secondArray" :key="index">{{item[pickerKey]}}</view>
				</picker-view-column>
			</picker-view>
			<!-- 三列选择-联动 -->
			<picker-view class="picker-view" indicator-style="height:100upx;" indicator-class="picker-view-selected-three" :value="pickerIndex" @change="pickerViewChangeThree"
			 v-if="mode=='three'">
				<picker-view-column>
					<view class="picker-item" v-for="(item, index) in pickerList" :key="index">{{item[pickerKey]}}</view>
				</picker-view-column>
				<picker-view-column>
					<view class="picker-item" v-for="(item, index) in pickerList[pickerIndex[0]].children" :key="index">{{item[pickerKey]}}</view>
				</picker-view-column>
				<picker-view-column>
					<view class="picker-item" v-for="(item, index) in pickerList[pickerIndex[0]].children[pickerIndex[1]].children"
					 :key="index">{{item[pickerKey]}}</view>
				</picker-view-column>
			</picker-view>
		</view>
	</view>
</template>

<script>
	// mode所有类型
	let pickerModeArray = ["one", "two_linkage", "two_nolinkage", "three"];

	export default {
		name: "wzp-picker",
		props: {
			// piker组件模式，默认单列
			mode: {
				type: String,
				required: true,
				// 校验数据
				validator: (value) => {
					return pickerModeArray.includes(value);
				}
			},
			// picker组件数组
			pickerList: {
				type: Array,
				required: true,
			},
			// picker组件数组
			pickerKey: {
				type: String,
				required: true,
			},
			// 默认picker选中项索引
			defaultIndex: {
				type: Array,
				required: true,
				validator: (value) => {
					if (value.length > 0 && value.length <= 3) {
						return true;
					}
					return false;
				}
			}
		},
		data() {
			return {
				pickerIndex: [], // picker索引值
				pickerShow: false
			};
		},
		created() {
			this.pickerIndex = this.defaultIndex;
		},
		watch: {
			// 匹配选中索引
			mode() {
				this.pickerIndex = this.defaultIndex;
			},
            defaultIndex() {
                this.pickerIndex = this.defaultIndex;
            }
		},
		methods: {
			// 单列选项变化
			pickerViewChangeOne(e) {
				this.pickerIndex = e.detail.value;
			},
			// 双列选项变化-包含联动和不联动
			pickerViewChangeTwo(e) {
				let changeValue = e.detail.value;
				if (this.mode == "two_linkage") {
					// 超规处理
					if (this.pickerList[changeValue[0]].children.length - 1 < changeValue[1]) {
						changeValue[1] = this.pickerList[changeValue[0]].children.length - 1;
					}
				}
				this.pickerIndex = changeValue;
			},
			// 三列联动选项变化
			pickerViewChangeThree(e) {
				let changeValue = e.detail.value;
				// 超规处理
				if (this.pickerList[changeValue[0]].children.length - 1 < changeValue[1]) {
					changeValue[1] = this.pickerList[changeValue[0]].children.length - 1;
				}
				if (this.pickerList[changeValue[0]].children[changeValue[1]].children.length - 1 < changeValue[2]) {
					changeValue[2] = this.pickerList[changeValue[0]].children[changeValue[1]].children.length - 1;
				}
				this.pickerIndex = changeValue;
			},
			// 显示组件
			showPicker() {
				// 隐藏软件盘
				uni.hideKeyboard();
				this.pickerShow = true;
			},
			// 确定事件——返回选中项的数组索引（也可以自定义其他返回数据，不过返回索引通用性更强）
			confirm() {
				this.pickerShow = false;
				this.$emit("onConfirm", this.pickerIndex);
			},
			// 隐藏组件
			closePicker() {
				this.pickerShow = false;
			}
		}
	}
</script>

<style>
	.picker-mask {
		position: fixed;
		top: 0;
		right: 0;
		left: 0;
		bottom: 0;
		z-index: 999;
		width: 100%;
		height: 100%;
		background-color: rgba(0, 0, 0, 0.3);
	}

	.picker-content {
		flex-direction: column;
		position: fixed;
		bottom: 0;
		left: 0;
		z-index: 1000;
		width: 100%;
		height: 600upx;
		background-color: #FFFFFF;
		transform: translateY(100%);
		transition: all 200ms ease;
	}

	.pickerShow {
		transform: translateY(0) !important;
	}

	.picker-content .picker-button {
		display: flex;
		justify-content: space-between;
		height: 80upx;
		line-height: 80upx;
	}

	.picker-button text {
		width: 180upx;
		font-size: 28upx;
		font-weight: 500;
		display: block;
		text-align: center;
	}

	.picker-content .picker-view {
		width: 100%;
		height: 500upx;
	}
	
	.picker-content .picker-view-selected-one,
	.picker-content .picker-view-selected-two,
	.picker-content .picker-view-selected-three {
		height: 100upx;
		line-height: 100upx;
		border-top: #0081FF 1upx solid;
		border-bottom: #0081FF 1upx solid;
	}

	.picker-content .picker-view-selected-one {
		position: relative;
		left: 25%;
		width: 50%;
	}

	.picker-content .picker-view-selected-two {
		position: relative;
		left: 15%;
		width: 70%;
	}

	.picker-content .picker-view-selected-three {
		position: relative;
		left: 5%;
		width: 90%;
	} 

	.picker-view .picker-item {
		width: 100%;
		height: 100upx;
		line-height: 100upx;
		font-size: 28upx;
		font-weight: 600;
		display: block;
		text-align: center;
	}
</style>
