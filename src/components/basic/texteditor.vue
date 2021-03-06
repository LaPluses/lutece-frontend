<template>
	<div v-show = "display" >
		<v-toolbar
			dense
			height = "40">
			<v-btn
				flat
				small
				@click = "previewContent">
				<v-icon
					v-show = "!preview"
					small> mdi-eye </v-icon>
				<v-icon
					v-show = "preview"
					small> mdi-eye-off </v-icon>
				<span class = "ml-2" > PREVIEW </span>
			</v-btn>
			<v-btn
				small
				flat
				@click = "triggerUpload">
				<input
					ref = "fileUploadInput"
					type = "file"
					style = "display:none;"
					enctype = "multipart/form-data"
					@change = "uploadFile">
				<v-icon small>mdi-image</v-icon>
				<span class = "ml-2" > UPLOAD </span>
			</v-btn>
			<v-spacer/>
			<a @click = "$emit('close-editor')" >
				<v-icon> mdi-close-circle </v-icon>
			</a>
		</v-toolbar>
		<v-textarea
			v-show = "!preview"
			ref = "content"
			v-model = "content"
			:loading = "uploading"
			class = "commentEditor"
			auto-grow
			solo
		/>
		<v-card v-if = "preview">
			<v-card-text>
				<async-mixrend-component
					:content = "content"
					class = "commentContent"
				/>
			</v-card-text>
		</v-card>
	</div>
</template>


<script>

import UploadImageGQL from '@/graphql/utils/uploadimage.gql';
import { getServerUri } from '@/utils';
import { AsyncMixrendComponent } from '@/components/async/mixrend/index';

export default {

	components: {
		AsyncMixrendComponent,
	},

	props: {
		display: {
			type: Boolean,
			required: true,
			default: false,
		},
	},
	data: () => ({
		content: '',
		preview: false,
		uploading: false,
	}),

	methods: {
		insertContent(value) {
			const textArea = this.$refs.content.$el.querySelector('textarea');
			const startPos = textArea.selectionStart;
			const endPos = textArea.selectionEnd;
			let cursorPos = startPos;
			const tmpStr = textArea.value;
			if (value === null) return;
			this.content = tmpStr.substring(0, startPos) + value
			+ tmpStr.substring(endPos, tmpStr.length);
			cursorPos += value.length;
			textArea.selectionStart = cursorPos;
			textArea.selectionEnd = cursorPos;
		},
		previewContent() {
			this.preview = !this.preview;
		},
		triggerUpload() {
			if (this.preview) {
				this.$store.commit('snackbar/setSnack', 'You can not upload image while previewing');
				return;
			}
			if (this.uploading) {
				this.$store.commit('snackbar/setSnack', 'Previous image is uploading, please wait a moment.');
				return;
			}
			this.$refs.fileUploadInput.click();
		},
		uploadFile(event) {
			const file = event.target.files[0];
			const maxsize = 2 * 1024 * 1024; // 2mb
			if (file.size > maxsize) {
				this.$store.commit('snackbar/setSnack', 'Image size should no more than 2mb.');
				return;
			}
			this.uploading = true;
			this.$apollo.mutate({
				mutation: UploadImageGQL,
				variables: {
					file,
				},
			})
				.then(response => response.data.UploadImage)
				.then((data) => {
					this.insertContent(`![](${getServerUri('http', data.path.slice(1))} =300x)`);
				})
				.catch((error) => {
					const af = JSON.parse(error.graphQLErrors[0].message);
					const msg = af.image['0'].message;
					this.$store.commit('snackbar/setSnack', msg);
				})
				.finally(() => { this.uploading = false; });
		},
	},
};
</script>

<style>
    .v-textarea.commentEditor textarea {
        font-size: 14px !important;
        resize: none;
        font-family: 'Monaco', courier, monospace !important;
		line-height: 1.5rem;
    }
</style>
