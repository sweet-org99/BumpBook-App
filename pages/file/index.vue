<template>
	<view class="u-wrap">
		<view class="u-menu-wrap">
			<scroll-view scroll-y scroll-with-animation class="u-tab-view menu-scroll-view" :scroll-top="scrollTop"
				:scroll-into-view="itemId">
				<view v-for="(item,index) in tabbar" :key="index" class="u-tab-item"
					:class="[current == index ? 'u-tab-item-active' : '']" @tap.stop="swichMenu(index)">
					<text class="u-line-1">{{item.checkDate}}</text>
				</view>
			</scroll-view>
			<scroll-view :scroll-top="scrollRightTop" scroll-y scroll-with-animation class="right-box"
				@scroll="rightScroll">
				<view class="page-view">
					<view class="class-item" :id="'item' + index" v-for="(item , index) in tabbar" :key="index">
						<view class="item-title">
							<text>{{item.checkDate}}</text>
						</view>
						<view class="align-center justify-center flex" style="margin-bottom: 10rpx;">
							<text class="title">病历单</text>
						</view>
						<view class="item-container">
							<view @longpress="delFile(item1)" class="thumb-box"
								v-for="(item1, index1) in item.bpBaseDetailInfoMap[1]" :key="index1">
								<image class="item-menu-image" :src="item1.url"
									@click="previewImage(item.bpBaseDetailInfoMap[1],index1)" mode=""></image>
								<!-- <view class="item-menu-name">{{item1.name}}</view> -->
							</view>
							<view class="thumb-box" @click="upload(item.id,1)">
								<image class="item-menu-image" src="@/static/images/file/upload.png" mode=""></image>
							</view>
						</view>
						<view class="logo-content align-center justify-center flex" style="margin-top: 20rpx;">
							<text class="title">检查结果</text>
						</view>
						<view class="item-container">
							<view @longpress="delFile(item1)" class="thumb-box"
								v-for="(item1, index1) in item.bpBaseDetailInfoMap[2]" :key="index1">
								<image class="item-menu-image" :src="item1.url"
									@click="previewImage(item.bpBaseDetailInfoMap[2],index1)" mode=""></image>
								<!-- <view class="item-menu-name">{{item1.name}}</view> -->
							</view>
							<view class="thumb-box" @click="upload(item.id,2)">
								<image class="item-menu-image" src="@/static/images/file/upload.png" mode=""></image>
							</view>
						</view>
						<view class="logo-content align-center justify-center flex" style="margin-top: 20rpx;">
							<text class="title">发票</text>
						</view>
						<view class="item-container">
							<view @longpress="delFile(item1)" class="thumb-box"
								v-for="(item1, index1) in item.bpBaseDetailInfoMap[3]" :key="index1">
								<image class="item-menu-image" :src="item1.url"
									@click="previewImage(item.bpBaseDetailInfoMap[3],index1)" mode=""></image>
								<!-- <view class="item-menu-name">{{item1.name}}</view> -->
							</view>
							<view class="thumb-box" @click="upload(item.id,3)">
								<image class="item-menu-image" src="@/static/images/file/upload.png" mode=""></image>
							</view>
						</view>
					</view>
				</view>
			</scroll-view>
		</view>
		<view>
			<uni-fab ref="fab" :pattern="pattern" :content="content" :horizontal="horizontal" :vertical="vertical"
				:popMenu='popMenu' :direction="direction" @trigger="trigger" @fabClick="fabClick" type='folder-add' />
		</view>
		<view>
			<view>
				<uni-calendar ref="calendar" :insert="false" :lunar="false" :selected='calendarSelected'
					@confirm="calendarChange" />
			</view>
		</view>
	</view>
</template>
<script>
	import config from '@/config'
	import {
		uploadFile,
		getDataList,
		addBpBaseInfo,
		deleteBpBaseInfo,
		deleteBpBaseDetailInfo
	} from '@/api/myda/file'
	import {
		getToken
	} from '@/utils/auth'
	export default {
		data() {
			return {
				scrollTop: 0, //tab标题的滚动条位置
				oldScrollTop: 0,
				current: 0, // 预设当前项的值
				menuHeight: 0, // 左边菜单的高度
				menuItemHeight: 0, // 左边菜单item的高度
				itemId: '', // 栏目右边scroll-view用于滚动的id
				tabbar: [],
				menuItemPos: [],
				arr: [],
				scrollRightTop: 0, // 右边栏目scroll-view的滚动条高度
				timer: null, // 定时器
				show: false,
				horizontal: 'right',
				vertical: 'bottom',
				popMenu: false,
				pattern: {
					buttonColor: '#959aff',
					icon: 'folder-add-filled'
				},
				calendarSelected: [],
			}
		},
		onLoad() {
			this.getBpList()
		},
		onReady() {
			this.getMenuItemTop()
		},
		methods: {
			onShow() {
				if (this.$store.state.user.deptId == '') {
					this.$tab.reLaunch('/pages/mine/index')
				}
			},
			// 点击左边的栏目切换
			async swichMenu(index) {
				if (this.arr.length == 0) {
					await this.getMenuItemTop();
				}
				if (index == this.current) return;
				this.scrollRightTop = this.oldScrollTop;
				this.$nextTick(function() {
					this.scrollRightTop = this.arr[index];
					this.current = index;
					this.leftMenuStatus(index);
				})
			},
			// 获取一个目标元素的高度
			getElRect(elClass, dataVal) {
				new Promise((resolve, reject) => {
					const query = uni.createSelectorQuery().in(this);
					query.select('.' + elClass).fields({
						size: true
					}, res => {
						// 如果节点尚未生成，res值为null，循环调用执行
						if (!res) {
							setTimeout(() => {
								this.getElRect(elClass);
							}, 10);
							return;
						}
						this[dataVal] = res.height;
						resolve();
					}).exec();
				})
			},
			// 观测元素相交状态
			async observer() {
				this.tabbar.map((val, index) => {
					let observer = uni.createIntersectionObserver(this);
					// 检测右边scroll-view的id为itemxx的元素与right-box的相交状态
					// 如果跟.right-box底部相交，就动态设置左边栏目的活动状态
					observer.relativeTo('.right-box', {
						top: 0
					}).observe('#item' + index, res => {
						if (res.intersectionRatio > 0) {
							let id = res.id.substring(4);
							this.leftMenuStatus(id);
						}
					})
				})
			},
			// 设置左边菜单的滚动状态
			async leftMenuStatus(index) {
				this.current = index;
				// 如果为0，意味着尚未初始化
				if (this.menuHeight == 0 || this.menuItemHeight == 0) {
					await this.getElRect('menu-scroll-view', 'menuHeight');
					await this.getElRect('u-tab-item', 'menuItemHeight');
				}
				// 将菜单活动item垂直居中
				this.scrollTop = index * this.menuItemHeight + this.menuItemHeight / 2 - this.menuHeight / 2;
			},
			// 获取右边菜单每个item到顶部的距离
			getMenuItemTop() {
				new Promise(resolve => {
					let selectorQuery = uni.createSelectorQuery();
					selectorQuery.selectAll('.class-item').boundingClientRect((rects) => {
						// 如果节点尚未生成，rects值为[](因为用selectAll，所以返回的是数组)，循环调用执行
						if (!rects.length) {
							setTimeout(() => {
								this.getMenuItemTop();
							}, 10);
							return;
						}
						rects.forEach((rect) => {
							// 这里减去rects[0].top，是因为第一项顶部可能不是贴到导航栏(比如有个搜索框的情况)
							this.arr.push(rect.top - rects[0].top);
							resolve();
						})
					}).exec()
				})
			},
			// 右边菜单滚动
			async rightScroll(e) {
				this.oldScrollTop = e.detail.scrollTop;
				if (this.arr.length == 0) {
					await this.getMenuItemTop();
				}
				if (this.timer) return;
				if (!this.menuHeight) {
					await this.getElRect('menu-scroll-view', 'menuHeight');
				}
				setTimeout(() => { // 节流
					this.timer = null;
					// scrollHeight为右边菜单垂直中点位置
					let scrollHeight = e.detail.scrollTop + this.menuHeight / 2;
					for (let i = 0; i < this.arr.length; i++) {
						let height1 = this.arr[i];
						let height2 = this.arr[i + 1];
						// 如果不存在height2，意味着数据循环已经到了最后一个，设置左边菜单为最后一项即可
						if (!height2 || scrollHeight >= height1 && scrollHeight < height2) {
							this.leftMenuStatus(i);
							return;
						}
					}
				}, 10)
			},
			upload(baseInfoId, type) {
				uni.chooseImage({
					success: (chooseImageRes) => {
						const tempFilePaths = chooseImageRes.tempFilePaths;
						for (const filePath of tempFilePaths) {
							let data = {
								name: 'file',
								filePath: filePath,
								formData: {
									type: type,
									baBaseId: baseInfoId
								}
							}
							uploadFile(data).then(response => {
								this.getBpList()
							})

						}

					}
				})
			},
			getBpList() {
				this.tabbar = []
				getDataList().then(res => {
					if (res.data != undefined) {
						this.tabbar = res.data
					}
				})

			},
			fabClick() {
				// this.$refs.popup.open('center')
				this.$refs.calendar.open();
			},
			popupChange() {
				console.log('=======')
			},
			calendarChange(e) {
				//新增日期
				let data = {
					checkDate: e.fulldate
				}
				addBpBaseInfo(data).then(res => {
					uni.showToast({
						title: res.msg,
						duration: 2000
					})
					this.getBpList()
				})
			},
			previewImage(item, index) {
				let urls = []
				item.forEach(e => {
					urls.push(e.url)
				})
				// 预览图片
				uni.previewImage({
					urls: urls,
					current: index,
					longPressActions: {
						itemList: ['发送给朋友', '保存图片', '收藏'],
						success: function(data) {
							console.log('选中了第' + (data.tapIndex + 1) + '个按钮,第' + (data.index + 1) + '张图片');
						},
						fail: function(err) {
							console.log(err.errMsg);
						}
					}
				});
			},
			delFile(item) {
				uni.showModal({
					title: '提示',
					content: '确认删除此照片？',
					success: (res) => {
						if (res.confirm) {
							this.delDetailFile(item.id)
						} else if (res.cancel) {
							console.log('用户已取消操作！')
						}
					}
				})
			},
			delBaseInfo(id) {
				deleteBpBaseInfo(id).then(res => {
					uni.showToast({
						title: '操作成功！',
						icon: 'success'
					});
					this.getBpList()
				})
			},
			delDetailFile(fileId) {
				deleteBpBaseDetailInfo(fileId).then(res => {
					uni.showToast({
						title: '操作成功！',
						icon: 'success'
					});
					this.getBpList()
				})
			}
		}
	}
</script>

<style lang="scss" scoped>
	.u-wrap {
		height: calc(100vh);
		/* #ifdef H5 */
		height: calc(100vh - var(--window-top));
		/* #endif */
		display: flex;
		flex-direction: column;
	}

	.u-search-box {
		padding: 18rpx 30rpx;
	}

	.u-menu-wrap {
		flex: 1;
		display: flex;
		overflow: hidden;
	}

	.u-search-inner {
		background-color: rgb(234, 234, 234);
		border-radius: 100rpx;
		display: flex;
		align-items: center;
		padding: 10rpx 16rpx;
	}

	.u-search-text {
		font-size: 26rpx;
		color: $u-tips-color;
		margin-left: 10rpx;
	}

	.u-tab-view {
		width: 200rpx;
		height: 100%;
	}

	.u-tab-item {
		height: 110rpx;
		background: #f6f6f6;
		box-sizing: border-box;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 26rpx;
		color: #444;
		font-weight: 400;
		line-height: 1;
	}

	.u-tab-item-active {
		position: relative;
		color: #000;
		font-size: 30rpx;
		font-weight: 600;
		background: #fff;
	}

	.u-tab-item-active::before {
		content: "";
		position: absolute;
		border-left: 4px solid $u-type-primary;
		height: 32rpx;
		left: 0;
		top: 39rpx;
	}

	.u-tab-view {
		height: 100%;
	}

	.right-box {
		background-color: rgb(250, 250, 250);
	}

	.page-view {
		padding: 16rpx;
	}

	.class-item {
		margin-bottom: 30rpx;
		background-color: #fff;
		padding: 16rpx;
		border-radius: 8rpx;
	}

	.class-item:last-child {
		min-height: 100vh;
	}

	.item-title {
		font-size: 26rpx;
		color: $u-main-color;
		font-weight: bold;
	}

	.item-menu-name {
		font-weight: normal;
		font-size: 24rpx;
		color: $u-main-color;
	}

	.item-container {
		display: flex;
		flex-wrap: wrap;
	}

	.thumb-box {
		width: 33.333333%;
		display: flex;
		align-items: center;
		justify-content: center;
		flex-direction: column;
		margin-top: 20rpx;
	}

	.item-menu-image {
		width: 120rpx;
		height: 120rpx;
	}

	.title {
		margin-left: 10px;
		color: #A0A0A0
	}

	// .warp {
	// 		padding: 10px;
	// 	}

	// .button {
	// 	margin-bottom: 10px;
	// }
</style>