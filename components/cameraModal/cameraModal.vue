<template>
	<view class="root">
		<view class="cu-modal show">
			<view class="cu-dialog">
				<view class="cu-bar bg-white justify-end">
					<view class="content">{{$t('components.tips.title')}}</view>
					<view class="action" @tap="ChooseImage(false)">
						<text class="cuIcon-close text-red"></text>
					</view>
				</view>
				<view class="padding-xl">
					{{$t('tabBar.share.uploadMethod')}}
				</view>
				<view class="cu-bar bg-white">
					<view class="action margin-0 flex-sub text-blue solid-left" @tap="ChooseImage(true,0)">{{$t('tabBar.share.camera')}}</view>
					<view class="action margin-0 flex-sub text-blue solid-left" @tap="ChooseImage(true,1)">{{$t('tabBar.share.album')}}</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		methods: {
			ChooseImage(flag, index) {
				this.$emit('ChooseImage', {
					flag,
					index
				})
			}
		}
	}
</script>

<style>
	page {
		height: auto;
		min-height: 100%;
	}
</style>
