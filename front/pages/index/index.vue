<template>
	<view class="page">
		<view class="header">
			<input v-model="title" class="title-input" placeholder="标题" />
			<button class="save-btn" size="mini" type="primary" :loading="saving" @tap="saveContent">保存</button>
			<button class="new-btn" size="mini" @tap="newDoc">新建</button>
		</view>

		<view class="toolbar">
			<view class="tool" @tap="format('bold')">粗体</view>
			<view class="tool" @tap="format('italic')">斜体</view>
			<view class="tool" @tap="format('underline')">下划线</view>
			<view class="tool red" @tap="format('color', '#e53935')">红字</view>
			<view class="tool" @tap="removeFormat">清除格式</view>
			<view class="tool" @tap="insertDivider">分割线</view>
			<view class="tool" @tap="insertImage">图片</view>
		</view>

		<editor id="editor" class="editor" placeholder="请输入内容..." @ready="onEditorReady" @input="onEditorInput"></editor>

		<view class="preview-label">预览</view>
		<mp-html :content="html" :preview-img="true" />

		<view class="list">
			<view class="list-title">富文本列表</view>
			<view v-if="docs.length === 0" class="empty">暂无记录</view>
			<view v-else class="list-item" v-for="item in docs" :key="item.id">
				<view class="item-main" @tap="loadDoc(item.id)">
					<view class="item-title">{{ item.title || '未命名' }}</view>
					<view class="item-meta">{{ item.updatedAt || item.createdAt || '' }}</view>
				</view>
				<view class="item-actions">
					<button class="item-btn" size="mini" @tap.stop="loadDoc(item.id)">编辑</button>
					<button class="item-btn danger" size="mini" @tap.stop="deleteDoc(item.id)">删除</button>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	import MpHtml from '@/uni_modules/mp-html/components/mp-html/mp-html.vue'

	export default {
		components: {
			MpHtml
		},
		data() {
			return {
				title: '',
				html: '',
				editorCtx: null,
				saving: false,
				uploadUrl: 'http://127.0.0.1:9873/api/upload',
				baseUrl: 'http://127.0.0.1:9873/api/rich-text',
				docs: [],
				currentId: null
			}
		},
		onLoad() {
			this.fetchList()
		},
		methods: {
			onEditorReady() {
				uni.createSelectorQuery()
					.in(this)
					.select('#editor')
					.context((res) => {
						this.editorCtx = res.context
						if (this.html) {
							this.setEditorHtml(this.html)
						}
					})
					.exec()
			},
			onEditorInput(e) {
				this.html = e.detail.html || ''
			},
			fetchList() {
				uni.request({
					url: this.baseUrl,
					method: 'GET',
					success: (res) => {
						const payload = res.data || {}
						if (payload.success) {
							this.docs = payload.data || []
						} else {
							this.docs = []
						}
					},
					fail: () => {
						this.docs = []
					}
				})
			},
			loadDoc(id) {
				if (!id) {
					return
				}
				uni.request({
					url: `${this.baseUrl}/${id}`,
					method: 'GET',
					success: (res) => {
						const payload = res.data || {}
						const doc = payload.data
						if (!doc) {
							uni.showToast({ title: '记录不存在', icon: 'none' })
							return
						}
						this.currentId = doc.id
						this.title = doc.title || ''
						this.html = doc.contentHtml || ''
						this.setEditorHtml(this.html)
					}
				})
			},
			deleteDoc(id) {
				if (!id) {
					return
				}
				uni.showModal({
					title: '确认删除',
					content: '删除后不可恢复，是否继续？',
					success: (res) => {
						if (!res.confirm) {
							return
						}
						uni.request({
							url: `${this.baseUrl}/${id}`,
							method: 'DELETE',
							success: (resp) => {
								const payload = resp.data || {}
								if (payload.success) {
									if (this.currentId === id) {
										this.newDoc()
									}
									this.fetchList()
									uni.showToast({ title: '已删除', icon: 'success' })
								} else {
									uni.showToast({ title: payload.message || '删除失败', icon: 'none' })
								}
							}
						})
					}
				})
			},
			newDoc() {
				this.currentId = null
				this.title = ''
				this.html = ''
				this.setEditorHtml('')
			},
			format(name, value) {
				if (!this.editorCtx) {
					return
				}
				this.editorCtx.format(name, value)
			},
			removeFormat() {
				if (!this.editorCtx) {
					return
				}
				this.editorCtx.removeFormat()
			},
			insertDivider() {
				if (!this.editorCtx) {
					return
				}
				this.editorCtx.insertDivider()
			},
			insertImage() {
				if (!this.editorCtx) {
					return
				}
				uni.chooseImage({
					count: 1,
					success: (res) => {
						const [filePath] = res.tempFilePaths
						this.uploadImage(filePath)
					}
				})
			},
			uploadImage(filePath) {
				uni.uploadFile({
					url: this.uploadUrl,
					filePath,
					name: 'file',
					success: (res) => {
						let payload = {}
						try {
							payload = JSON.parse(res.data || '{}')
						} catch (err) {
							payload = {}
						}
						const url = (payload.data && payload.data.url) || payload.url
						if (!url) {
							uni.showToast({ title: '上传失败', icon: 'none' })
							return
						}
						this.editorCtx.insertImage({
							src: url,
							width: '100%'
						})
					},
					fail: () => {
						uni.showToast({ title: '上传失败', icon: 'none' })
					}
				})
			},
			setEditorHtml(html) {
				if (!this.editorCtx) {
					return
				}
				this.editorCtx.setContents({
					html: html || ''
				})
			},
			saveContent() {
				if (!this.editorCtx) {
					return
				}
				this.saving = true
				this.editorCtx.getContents({
					success: (res) => {
						this.html = res.html || ''
						uni.request({
							url: this.baseUrl,
							method: 'POST',
							data: {
								id: this.currentId,
								title: this.title,
								html: res.html || ''
							},
							success: (resp) => {
								const payload = resp.data || {}
								if (payload.success && payload.data) {
									this.currentId = payload.data
								}
								if (!payload.success) {
									uni.showToast({ title: payload.message || '保存失败', icon: 'none' })
								} else {
									uni.showToast({ title: '已保存', icon: 'success' })
								}
								this.fetchList()
							},
							complete: () => {
								this.saving = false
							}
						})
					},
					fail: () => {
						this.saving = false
					}
				})
			}
		}
	}
</script>

<style>
	.page {
		padding: 16px;
		background: #f6f6f6;
		min-height: 100vh;
		box-sizing: border-box;
	}

	.header {
		display: flex;
		align-items: center;
		gap: 8px;
	}

	.title-input {
		flex: 1;
		background: #fff;
		border: 1px solid #e0e0e0;
		border-radius: 6px;
		padding: 8px 10px;
		font-size: 14px;
	}

	.save-btn {
		margin: 0;
	}

	.new-btn {
		margin: 0;
	}

	.toolbar {
		display: flex;
		flex-wrap: wrap;
		gap: 8px;
		margin-top: 12px;
	}

	.tool {
		padding: 6px 10px;
		border: 1px solid #d9d9d9;
		border-radius: 6px;
		background: #fff;
		font-size: 12px;
	}

	.tool.red {
		color: #e53935;
		border-color: #e53935;
	}

	.editor {
		min-height: 280px;
		margin-top: 12px;
		padding: 10px;
		background: #fff;
		border: 1px solid #e0e0e0;
		border-radius: 6px;
		line-height: 1.5;
	}

	.preview-label {
		margin: 16px 0 8px;
		font-weight: 600;
	}

	.list {
		margin-top: 16px;
	}

	.list-title {
		font-weight: 600;
		margin-bottom: 8px;
	}

	.empty {
		color: #999;
		font-size: 12px;
		padding: 8px 0;
	}

	.list-item {
		display: flex;
		align-items: center;
		justify-content: space-between;
		background: #fff;
		border: 1px solid #e0e0e0;
		border-radius: 6px;
		padding: 8px 10px;
		margin-bottom: 8px;
	}

	.item-main {
		flex: 1;
		padding-right: 8px;
	}

	.item-title {
		font-size: 14px;
		font-weight: 500;
	}

	.item-meta {
		font-size: 11px;
		color: #999;
		margin-top: 2px;
	}

	.item-actions {
		display: flex;
		gap: 6px;
	}

	.item-btn {
		margin: 0;
	}

	.item-btn.danger {
		color: #e53935;
	}
</style>
