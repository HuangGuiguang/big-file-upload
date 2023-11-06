<template>
    <div>
        <!-- <input type="file" name="" id="" @change="handleFileChange">
        <el-button @click="handleUpload">upload</el-button> -->

        <div>
            <input type="file" @change="handleFileChange" />
            <el-button @click="handleUpload">upload</el-button>
            <!-- <el-button @click="handleResume" v-if="status === Status.pause">resume</el-button>
            <el-button v-else :disabled="status !== Status.uploading || !container.hash"
                @click="handlePause">pause</el-button>
            <el-button @click="handleDelete">delete</el-button> -->
        </div>
        <div>
            <div>
                <div>calculate chunk hash</div>
                <el-progress :percentage="hashPercentage"></el-progress>
            </div>
            <!-- <div>
                <div>percentage</div>
                <el-progress :percentage="fakeUploadPercentage"></el-progress>
            </div> -->
        </div>
        <el-table :data="data">
            <el-table-column prop="hash" label="chunk hash" align="center"></el-table-column>
            <el-table-column label="size(KB)" align="center" width="120">
                <template v-slot="{ row }">
                    {{ row.size | transformByte }}
                </template>
            </el-table-column>
            <el-table-column label="percentage" align="center">
                <template v-slot="{ row }">
                    <el-progress :percentage="row.percentage" color="#909399"></el-progress>
                </template>
            </el-table-column>
        </el-table>
    </div>
</template>

<script>
// 切片大小
// the chunk size, 10M
const SIZE = 10 * 1024 * 1024;
export default {
    filters: {
        transformByte(val) {
            return Number((val / 1024).toFixed(0));
        }
    },
    data() {
        return {
            container: {
                file: null,
                worker: null,
                hash: ''
            },
            data: [],
            hashPercentage: 0
        }
    },
    computed: {
        uploadPercentage() {
            if (!this.container.file || !this.data.length) return 0;
            const loaded = this.data
                .map(item => item.size * item.percent)
                .reduce((acc, cur) => acc + cur);
            return parseInt((loaded / this.container.file.size).toFixed(2));
        }
    },
    methods: {
        handleFileChange(e) {
            // 拿到选择的文件
            const [file] = e.target.files;
            if (!file) return;
            Object.assign(this.$data, this.$options.data())
            this.container.file = file
        },
        /**
         * 处理文件切片
         * @param {*} file 文件
         * @param {*} size 切片大小
         */
        createFileChunk(file, size = SIZE) {
            const fileChunkList = []
            let cur = 0
            while (cur < file.size) {
                fileChunkList.push({ file: file.slice(cur, cur + size) })
                cur += size
            }
            return fileChunkList
        },
        /**
         * 并发上传切片
         */
        async uploadChunks() {
            const requestList = this.data.map(({ chunk, hash, index, fileHash }) => {
                const formData = new FormData()
                formData.append("chunk", chunk) // 实际的文件
                formData.append('hash', hash) // 记录文件索引
                formData.append("filename", this.container.file.name)
                formData.append("fileHash", fileHash)
                return { formData, index }
            }).map(({ formData }, index) => this.request({
                url: "http://localhost:3000",
                data: formData,
                onProgress: this.createProgressHandler(this.data[index]),
            })
            )
            // 并发请求
            await Promise.all(requestList)
            // 合并切片
            await this.mergeRequest()
        },
        async handleUpload() {
            if (!this.container.file) return
            const fileChunkList = this.createFileChunk(this.container.file)
            this.container.hash = await this.calculateHash(fileChunkList)
            console.log(this.container.hash)
            this.data = fileChunkList.map(({ file }, index) => {
                return {
                    fileHash: this.container.hash,
                    index,
                    chunk: file,
                    hash: this.container.file.name + "-" + index,
                    size: file.size,
                    percent: 0 // 记录上传的进度
                }
            })
            await this.uploadChunks()
        },
        calculateHash(fileChunkList) {
            return new Promise(resolve => {
                // 添加worker
                this.container.worker = new Worker("/hash.js")
                this.container.worker.postMessage({ fileChunkList })
                this.container.worker.onmessage = e => {
                    const { percentage, hash } = e.data
                    this.hashPercentage = percentage
                    if (hash) {
                        resolve(hash)
                    }
                }
            })
        },
        createProgressHandler(item) {
            return e => {
                item.percent = parseInt(String(e.loaded / e.total) * 100)
            }
        },
        async mergeRequest() {
            let _this = this
            await this.request({
                url: "http://localhost:3000/merge",
                headers: {
                    "content-type": "application/json"
                },
                data: JSON.stringify({
                    size: SIZE,
                    filename: _this.container.file.name
                })
            })
        },
        /**
         * 自定义请求, 使用原生的XMLHttpRequest
         * @param {*} param0 
         */
        request({
            url,
            method = "post",
            data,
            headers = {},
            onProgress
        }) {
            return new Promise(resolve => {
                const xhr = new XMLHttpRequest();
                xhr.upload.onprogress = onProgress // 绑定process进度函数
                xhr.open(method, url);
                Object.keys(headers).forEach(key =>
                    xhr.setRequestHeader(key, headers[key])
                );
                xhr.send(data);
                xhr.onload = e => {
                    resolve({
                        data: e.target.response
                    });
                };
            });
        }
    },
}
</script>