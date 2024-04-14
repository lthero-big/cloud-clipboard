<template>
    <div>
        <div class="headline text--primary mb-4">发送文本</div>
        <v-textarea
            outlined
            dense
            rows="6"
            :counter="$root.config.text.limit"
            placeholder="请输入需要发送的文本"
            v-model="$root.send.text"
        ></v-textarea>
        <div v-html="renderedContent" class="rendered-markdown"></div>
        <div class="text-right">
            <v-btn
                color="primary"
                :block="$vuetify.breakpoint.smAndDown"
                :disabled="!$root.send.text || !$root.websocket || $root.send.text.length > $root.config.text.limit"
                @click="send"
            >发送</v-btn>
        </div>
    </div>
</template>

<style>
.rendered-markdown {
    max-height: 300px; /* 设置最大高度，超过这个高度的内容将通过滚动条显示 */
    overflow-y: auto; /* 垂直方向上，如果内容超出则显示滚动条 */
    padding: 10px; /* 可选，为内容添加一些内边距 */
    border: 1px solid #ccc; /* 可选，为容器添加边框，使之更清晰 */
    background-color: #f9f9f9; /* 可选，设置背景颜色 */
    margin-bottom: 10px; /* 可选，为容器添加下外边距 */
}
</style>

<script>
import MarkdownIt from 'markdown-it';
export default {
    name: 'send-text',
    data() {
        return {
            md: new MarkdownIt(),
            renderedContent: '',
        };
    },
    watch: {
        // 当文本变化时，重新渲染 Markdown
        '$root.send.text': function(newVal) {
            this.renderedContent = this.md.render(newVal);
        }
    },
    methods: {
        send() {
            this.$http.post(
                '/text',
                this.$root.send.text,
                {
                    params: new URLSearchParams([['room', this.$root.room]]),
                    headers: {
                        'Content-Type': 'text/plain',
                    },
                },
            ).then(response => {
                this.$toast('发送成功');
                this.$root.send.text = '';
            }).catch(error => {
                if (error.response && error.response.data.msg) {
                    this.$toast(`发送失败：${error.response.data.msg}`);
                } else {
                    this.$toast('发送失败');
                }
            });
        },
    },
}

</script>
