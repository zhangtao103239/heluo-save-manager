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
                        <el-collapse-item title="地区好感度" name="areaFriendly">
                            <el-row>
                                <el-col :sm="12" :md="6" v-for="(_, name) in saveJson.AreaFriendly"
                                        :key="name">
                                    <el-form-item :label="name">
                                        <el-input v-model="saveJson.AreaFriendly[name]"></el-input>
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
                                    <el-button type="warning" @click="removeStolen" style="width: 8rem">一键洗白</el-button>
                                </el-col>
                            </el-row>
                        </el-collapse-item>
                        <el-collapse-item title="队友管理" name="teammates">
                            <el-table stripe border :data="teammates">
                                <el-table-column
                                        prop="Id"
                                        label="NpcId">
                                </el-table-column>
                                <el-table-column
                                        label="姓名">
                                    <template slot-scope="scope">
                                        <span>{{npc[scope.row.Id]}}</span>
                                    </template>
                                </el-table-column>
                                <el-table-column label="等级">
                                    <template slot-scope="scope">
                                        <el-input-number v-model="scope.row.Level"></el-input-number>
                                    </template>
                                </el-table-column>
                                <el-table-column label="感悟点">
                                    <template slot-scope="scope">
                                        <el-input-number v-model="scope.row.TalentData.NewPoint"></el-input-number>
                                    </template>
                                </el-table-column>
                                <el-table-column label="操作">
                                    <template slot-scope="scope">
                                        <el-row>
                                            <el-col :span="12">
                                                <el-button type="primary"
                                                           @click="editTeammates(scope.row)">修改
                                                </el-button>
                                            </el-col>
                                            <el-col :span="12">
                                                <el-button type="danger"
                                                           :disabled="scope.row.Id === 'Player'"
                                                           @click="deleteTeammates(scope.row)">删除
                                                </el-button>
                                            </el-col>
                                        </el-row>
                                    </template>
                                </el-table-column>
                            </el-table>
                            <el-button style="width: 200px" @click="addNewTeammate">新增队友</el-button>
                        </el-collapse-item>
                        <el-collapse-item title="其他修改" name="others">
                            <el-row>
                                <el-col :sm="12" :md="6">
                                    <el-form-item label="姓">
                                        <el-input v-model="saveJson.PlayerSurname"></el-input>
                                    </el-form-item>
                                </el-col>
                                <el-col :sm="12" :md="6">
                                    <el-form-item label="名">
                                        <el-input v-model="saveJson.PlayerName"></el-input>
                                    </el-form-item>
                                </el-col>
                                <el-col :sm="12" :md="6">
                                    <el-form-item label="难度">
                                        <el-select v-model="saveJson.Difficulty" placeholder="请选择">
                                            <el-option label="简单" :value="0">
                                            </el-option>
                                            <el-option label="普通" :value="1">
                                            </el-option>
                                            <el-option label="困难" :value="2">
                                            </el-option>
                                            <el-option label="炼狱" :value="3">
                                            </el-option>
                                        </el-select>
                                    </el-form-item>
                                </el-col>
                            </el-row>
                        </el-collapse-item>
                    </el-collapse>
                    <el-button @click="downloadSaver" type="success">下载</el-button>
                </el-form>
            </div>
        </el-card>
        <el-dialog :title="npc[characterData.Id]+'基础数据修改'" :visible.sync="modifyDataShow" width="70rem">
            <el-form>
                <el-row :gutter="10">
                    <el-col :span="4" v-for="(value, name) in characterData.Data"
                            :key="name">
                        <el-form-item :label="translator[name]||name">
                            <el-input v-model="value.Base"></el-input>
                        </el-form-item>
                    </el-col>
                </el-row>
            </el-form>
        </el-dialog>
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
                                @click="addToInventory(scope.row)">新增
                        </el-button>
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
        <el-dialog title="选择队友" :visible.sync="teammatesChooseShow" width="40rem">
            <el-form label-width="5rem" label-position="left">
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
                        label="NpcId">
                </el-table-column>
                <el-table-column
                        prop="name"
                        label="名称">
                </el-table-column>
                <el-table-column label="操作">
                    <template slot-scope="scope">
                        <el-button
                                size="mini"
                                :disabled="saveJson.Teammates.some(value => value===scope.row.id)"
                                @click="() => saveJson.Teammates.push(scope.row.id)">新增
                        </el-button>
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
            if (columns.length >= 4) {
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
                    {id: "skillItem", name: "技能书"}],
                filePlace: '',
                fileName: '',
                fileTexts: [],
                characterData: {},
                waiting: false,
                itemChooseShow: false,
                teammatesChooseShow: false,
                modifyDataShow: false,
                saveJson: {},
                inventoryItem: readText2Object(inventoryItem),
                equipment: readText2Object(equipment),
                skillItem: readText2Object(skillItem),
                npc: npcItem,
                saveShow: ['teammates', 'others', 'areaFriendly'],
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
                return this.queriedData.slice((this.itemPage - 1) * 10, this.itemPage * 10)
            },
            teammates() {
                let teammate = []
                this.saveJson.Teammates.forEach(value => {
                    teammate.push(this.saveJson.Character[value])
                })
                return teammate
            },
            npcItem() {
                let npcItem = []
                Object.keys(this.npc).forEach(key => {
                    npcItem.push({
                        id: key,
                        name: this.npc[key]
                    })
                })
                return npcItem
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
                    this.saveShow.push('teammates')
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
                this.itemCategoryShow = 'inventoryItem'
                this.itemPage = 1
            },
            addToInventory(item) {
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
                    let hurtData = item.orial[14].split(/[()]/).filter(value => value !== '')
                    hurtData.forEach(value => {
                        let hurt = value.split(',')
                        itemData.Hurt[hurt[0]] = parseInt(hurt[1])
                    })
                    itemData.HurtDifference = item.orial[15]
                } else if (this.itemCategoryShow === 'skillItem') {
                    itemData.Level = item.orial[8]
                }
                // itemData.Durability = item.orial[]
                // console.log(itemData)
                this.saveJson.Character.Player.Inventory.push(itemData)
            },
            removeStolen() {
                this.saveJson.Character.Player.Inventory.forEach(value => {
                    value.Stolen = false
                })
                this.$message.success("洗白物品成功")
            },
            editTeammates(data) {
                this.modifyDataShow = true
                this.characterData = data
            },
            deleteTeammates(data) {
                let index = this.saveJson.Teammates.indexOf(data.Id)
                if (index !== -1) {
                    this.saveJson.Teammates.splice(index, 1)
                }
            },
            addNewTeammate() {
                this.teammatesChooseShow = true
                this.itemCategoryShow = 'npcItem'
                this.itemPage = 1
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
