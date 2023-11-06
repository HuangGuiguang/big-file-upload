<template>
    <div>
        <div>
            <input type="file" @change="handleFileChange" />
            <el-button @click="handleUpload">upload</el-button>
            <el-button @click="handleResume">Resume</el-button>
            <el-button @click="handlePause">pause</el-button>
        </div>
        <div>
            <div>
                <div>calculate chunk hash</div>
                <el-progress :percentage="hashPercentage"></el-progress>
            </div>
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
                    <el-progress :percentage="row.percent" color="#909399"></el-progress>
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
            hashPercentage: 0,
            requestList: [],
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
         * uploadedList: 已经上传了的文件切片列表
         */
        async uploadChunks(uploadedList) {
            const requestList = this.data
                .filter(({ hash }) => !uploadedList.includes(hash)) // 剔除已经上传了的文件切片
                .map(({ chunk, hash, index, fileHash }) => {
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
                    requestList: this.requestList
                })
                )
            // 并发请求
            await Promise.all(requestList)
            // 合并切片
            if (uploadedList.length + requestList.length === this.data.length) {
                await this.mergeRequest()
            }
        },
        async handleUpload() {
            if (!this.container.file) return
            const fileChunkList = this.createFileChunk(this.container.file)
            this.container.hash = await this.calculateHash(fileChunkList)
            // console.log(this.container.hash
            // 上传前请求确认该文件是否已经上传过
            const { shouldUpload, uploadedList } = await this.verifyUpload(this.container.file.name, this.container.hash)
            if (!shouldUpload) {
                this.$message.success("文件已经上传过, 上传成功")
                return
            }
            this.data = fileChunkList.map(({ file }, index) => {
                return {
                    fileHash: this.container.hash,
                    index,
                    chunk: file,
                    hash: this.container.hash + "-" + index,
                    size: file.size,
                    percent: uploadedList.includes(this.container.file.name + "-" + index) ? 100 : 0 // 已经上传的切片进度直接变成100%
                }
            })
            await this.uploadChunks(uploadedList)
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
                    filename: _this.container.file.name,
                    fileHash: this.container.hash,
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
            onProgress,
            requestList
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
                    if (requestList) {
                        const xhrIndex = requestList.findIndex(item => item === xhr)
                        requestList.splice(xhrIndex, 1)
                    }
                    resolve({
                        data: e.target.response
                    });
                };
                // 暴露当前xhr给外部
                requestList?.push(xhr)
            });
        },
        /**
         * 打断所有请求, 达到暂停的效果
         */
        handlePause() {
            this.requestList.forEach(xhr => xhr?.abort());
            this.requestList = [];
        },
        /**
         * 验证文件是否已经存在
         * @param {*} filename 
         * @param {*} fileHash 
         */
        async verifyUpload(filename, fileHash) {
            const { data } = await this.request({
                url: "http://localhost:3000/verify",
                headers: {
                    "content-type": "application/json"
                },
                data: JSON.stringify({
                    filename,
                    fileHash
                })
            })
            return JSON.parse(data)
        },
        async handleResume() {
            const { uploadedList } = await this.verifyUpload(
                this.container.file.name,
                this.container.hash
            );
            await this.uploadChunks(uploadedList);
        },
    },
}
</script>
