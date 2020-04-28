<template>
    <div id="app">
        <img src="./assets/logo.png"/>
        <h1>河洛群侠传存档修改器</h1>
        <el-form v-loading="waiting">
            <el-form-item label="选择你的存档文件">
                <el-input type="file" id="fileTest" accept=".save" v-model="filePlace"/>
            </el-form-item>
            <el-button @click="loadSaveFile" type="primary">解析</el-button>
        </el-form>
        <el-divider></el-divider>
        <el-card v-if="saveJson.PlayerName">
            <div slot="header">{{saveJson.PlayerSurname + saveJson.PlayerName}}的存档信息</div>
            <div>
                <el-form label-width="7rem">
                    <el-collapse v-model="saveShow">
                        <el-collapse-item title="主角基础数据" name="baseData">
                            <el-row>
                                <el-col :sm="12" :md="6" v-for="(value, name) in saveJson.Character.Player.Data"
                                        :key="name">
                                    <el-form-item :label="translator[name]||name">
                                        <el-input v-model="value.Base"></el-input>
                                    </el-form-item>
                                </el-col>
                            </el-row>
                        </el-collapse-item>
                        <el-collapse-item title="主角物品" name="inventory">
                            <el-row>
                                <el-col :sm="12" :md="6" v-for="(value, index) in saveJson.Character.Player.Inventory"
                                        :key="index">
                                    <el-form-item :label="getItem(value.ItemId).name"
                                                  :title="getItem(value.ItemId).desc">
                                        <el-input v-model="value.Count"></el-input>
                                    </el-form-item>
                                </el-col>
                            </el-row>
                            <el-row>
                                <el-col :span="12">
                                    <el-button type="primary" @click="showItemAddModal" style="width: 8rem">添加物品</el-button>
                                </el-col>
                                <el-col :span="12">
                                    <el-button type="info" @click="removeStolen" style="width: 8rem">一键洗白</el-button>
                                </el-col>

                            </el-row>
                        </el-collapse-item>
                    </el-collapse>
                    <el-button @click="downloadSaver" type="success">下载</el-button>
                </el-form>
            </div>
        </el-card>
        <el-dialog title="选择物品" :visible.sync="itemChooseShow" width="60rem">
            <el-form label-width="5rem" label-position="left">
                <el-form-item label="选择种类">
                    <el-radio-group v-model="itemCategoryShow">
                        <el-radio-button v-for="category in itemCategories" :key="category.id" :label="category.id">
                            {{category.name}}
                        </el-radio-button>
                    </el-radio-group>
                </el-form-item>

                <el-form-item label="查找">
                    <el-input placeholder="请输入内容"
                              v-model="itemFilterKey">
                        <i slot="prefix" class="el-input__icon el-icon-search"></i>
                    </el-input>
                </el-form-item>
            </el-form>
            <el-table
                    :data="pagedData" stripe border
                    style="width: 100%" max-height="450px">
                <el-table-column
                        prop="id"
                        label="ItemId"
                        width="80px">
                </el-table-column>
                <el-table-column
                        prop="name"
                        label="名称"
                        width="130px">
                </el-table-column>
                <el-table-column
                        prop="desc"
                        label="描述">
                </el-table-column>
                <el-table-column label="操作" width="80px">
                    <template slot-scope="scope">
                        <el-button
                                size="mini"
                                :disabled="saveJson.Character.Player.Inventory.some(value => value.ItemId===scope.row.id)"
                                @click="addToInventory(scope.row)">新增</el-button>
                    </template>
                </el-table-column>
            </el-table>
            <el-pagination
                    :current-page.sync="itemPage"
                    background
                    layout="prev, pager, next"
                    :total="queriedData.length">
            </el-pagination>
        </el-dialog>
    </div>
</template>

<script>
    import {saveAs} from 'file-saver'
    import equipment from "./data/equipment";
    import skillItem from './data/skill-item'
    import translator from "./data/translator";
    import inventoryItem from './data/inventory-item'
    import npcItem from './data/npc-item'

    function readText2Object(text) {
        let result = []
        text.split('\n').forEach(value => {
            let columns = value.split(/\t/)
            if (columns.length > 4 && columns[4] !== '0,') {
                let tmp = {}
                tmp.id = columns[0]
                tmp.name = columns[1]
                tmp.desc = columns[3]
                tmp.orial = columns
                result.push(tmp)
            }
        })
        return result
    }
    export default {
        name: 'app',
        components: {},
        data() {
            return {
                itemCategories: [
                    {id: "inventoryItem", name: "一般物品"},
                    {id: "equipment", name: "装备"},
                    {id: "skillItem", name: "技能书"}
                    ],
                filePlace: '',
                fileName: '',
                fileTexts: [],
                waiting: false,
                itemChooseShow: false,
                saveJson: {},
                inventoryItem: readText2Object(inventoryItem),
                equipment: readText2Object(equipment),
                skillItem: readText2Object(skillItem),
                npc: readText2Object(npcItem),
                saveShow: [],
                itemCategoryShow: 'inventoryItem',
                itemFilterKey: '',
                itemPage: 1,
                translator: translator
            }
        },
        computed: {
            queriedData() {
                let data = this[this.itemCategoryShow]
                data = data.filter(value => (value.name + value.desc).indexOf(this.itemFilterKey) !== -1)
                return data
            },
            pagedData() {
                return this.queriedData.slice((this.itemPage-1)*10, this.itemPage*10)
            }
        },
        methods: {
            loadSaveFile() {
                let file = document.getElementById("fileTest")
                this.fileName = file.files[0].name
                let reader = new FileReader()
                reader.onload = (event) => {
                    this.fileTexts = event.target.result.split('\n')
                    this.saveJson = JSON.parse(this.fileTexts[6])
                    this.saveShow.push('baseData')
                    this.saveShow.push('inventory')
                    this.waiting = false
                }
                this.waiting = true;
                setTimeout(() => reader.readAsText(file.files[0]), 20)
            },
            downloadSaver() {
                this.fileTexts[6] = JSON.stringify(this.saveJson)
                let blob = new Blob([this.fileTexts.join('\n')], {type: "text/plain;charset=utf-8"})
                saveAs(blob, this.fileName)
            },
            getItem(itemId) {
                let item = this.inventoryItem.find(value => value.id === itemId)
                    || this.equipment.find(value => value.id === itemId)
                    || this.skillItem.find(value => value.id === itemId)
                if (item) {
                    return item
                } else {
                    return {id: itemId, name: '未知'}
                }
            },
            showItemAddModal() {
                this.itemChooseShow = true
            },
            addToInventory (item) {
                let itemData = {
                    "ItemId": item.id,
                    "Count": 1,
                    "Durability": 0,
                    "MaxDurability": 0,
                    "Weight": 0,
                    "EffectId": [],
                    "IsNew": true,
                    "Stolen": false,
                    "Level": 0,
                    "Hurt": {},
                    "HurtDifference": 0.0,
                    "ForgeMaterials": {},
                    "QualityTitle": "",
                    "QuenchHoleCount": 0,
                    "QuenchEffect": [],
                    "ReforgeType": -1,
                    "Id": null
                }
                if (this.itemCategoryShow === 'equipment') {
                    itemData.Weight = item.orial[11]
                    itemData.MaxDurability = itemData.Durability = item.orial[12]
                    itemData.Level = item.orial[8]
                    itemData.QuenchHoleCount = 2
                    let hurtData = item.orial[14].split(/[()]/).filter(value => value!=='')
                    hurtData.forEach(value => {
                        let hurt = value.split(',')
                        itemData.Hurt[hurt[0]] = parseInt(hurt[1])
                    })
                    itemData.HurtDifference = item.orial[15]
                }
                // itemData.Durability = item.orial[]
                // console.log(itemData)
                this.saveJson.Character.Player.Inventory.push(itemData)
            },
            removeStolen () {
                this.saveJson.Character.Player.Inventory.forEach(value => {
                    value.Stolen = false
                })
            }
        }
        // mounted() {
        //     this.$http.get('inventoryitem.txt').then(result => {
        //         this.inventoryItem = readText2Object(result.data)
        //     })
        //     this.$http.get('npcitem.txt').then(result => {
        //         this.npc = readText2Object(result.data)
        //     })
        //     this.$http.get('equipment.txt').then(result => {
        //         this.equipment = readText2Object(result.data)
        //     })
        //     this.$http.get('skillitem.txt').then(result => {
        //         this.skillItem = readText2Object(result.data)
        //     })
        //     this.$http.get('translator.json').then(result => {
        //         this.translator = result.data
        //     })
        // }
    }
</script>

<style>
    #app {
        font-family: 'Avenir', Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        margin-top: 60px;
    }
</style>
