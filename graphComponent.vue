<template>
    <div class="graph-main">
        <div :style="graphMainStyle" class="graph-main-container" ref="main">
            <div class="graph-header">
                <div class="graph-header-item" v-for="(item,index) in titleData" :key="item.id" :id="index" :ref="'header_'+index" :style="{flex:item.size,minWidth:titleWidth+'px'}">
                    <div class="graph-header-item-title">{{item.name}}</div>
                    <div class="desc">{{item.desc}}</div>
                </div>
            </div>

            <div class="graph-content" ref="graphContent">
                <div class="graph-content-main" :style="{height:praghHeight}">
                    <div ref="graph" class="graph-main-items" id="graphMain"></div>

                    <svg ref="svg" class="graph-svg">
                        <defs>
                            <marker id="triangle" viewBox="0 0 40 40" refX="0" refY="40" markWidth="40" markHeight="40" fill="red" orient="auto">
                                <path d="M 0 0 L 40 20 L 0 40 Z" />
                            </marker>
                            <marker id="arrowEnd" markerHeight="9" markerWidth="9" refX="8" refY="3" fill="#1eaadc" viewBox="0 0 9 6" orient="auto-start-reverse" markerUnits="userSpaceOnUse">
                                <path d="M0,0 L0,6 L9,3 z" />
                            </marker>
                        </defs>
                    </svg>
                </div>
            </div>
        </div>

        <a-modal title="推演图片" :visible="imagesModal" :footer="null" :width="800" @cancel="imagesModal=false">
            <img :src="images" alt class="graph-images" />
        </a-modal>
    </div>
</template>

<script>
export default {
    data() {
        return {
            imagesModal: false,
            images: require("@/assets/images/graph.png"),
            scale: [1, 1],
            praghHeight: 0, //关系图高度
            titleWidth: 200, // 标题最小宽度设置 模块宽+连线第二个点x(30)*2
            itemHeight: 116, // 50, //每个模块高度 宽120
            itemMinWidth: 140, //每个模块最小宽度 120
            conWidth: 1000, //容器默认宽度
            headerHeight: 40 + 60, //表头高度+描述
            titleData: [],
            graphData: [],
            dependenciesData: [],
            renderPosition: [],
            diffLeft: 0,
            hasRender: false
        };
    },
    filters: {},
    methods: {
        /**
         * 渲染 header
         */
        renderTitle() { },
        /**
         * 绘制连线
         */
        createLine() {
            let svg = this.$refs.svg, // document.createElementNS("http://www.w3.org/2000/svg", "svg"),
                mainContainer = this.$refs.graphContent,
                mainContainerRect = mainContainer.getBoundingClientRect(),
                left = mainContainerRect.left,
                top = mainContainerRect.top;

            //添加svg
            for (let index = 0; index < this.dependenciesData.length; index++) {
                const element = this.dependenciesData[index];
                //查找元素点
                let fromEle = document.getElementById("item_" + element.from),
                    toEle = document.getElementById("item_" + element.to);
                if (!fromEle || !toEle) continue;
                fromEle = fromEle.getBoundingClientRect();
                toEle = toEle.getBoundingClientRect();

                let x1 = fromEle.right - left,
                    y1 = fromEle.top + fromEle.height / 2 - top,
                    x2 = toEle.left - left,
                    y2 = toEle.top + toEle.height / 2 - top;
                //连线
                let line = this.drawDependenciesPolyline(x1, y1, x2, y2, svg, toEle);
                // <line x1="434" y1="245" x2="489" y2="489" id="line_xyj_e" stroke-width="1" stroke="green" class="removeInfo"></line>

                svg.appendChild(line);
            }
        },
        /**
         * 绘制折线
         */
        drawDependenciesPolyline(x1, y1, x2, y2, svg, toEle) {
            let line = document.createElementNS(
                "http://www.w3.org/2000/svg",
                "polyline"
            );
            let points = [x1, y1],
                breakPoints = [],
                crossDomain =
                    x2 - x1 > this.$refs.header_0[0].getBoundingClientRect().width, //连线点是否跨域
                minSpacing = y2 - y1 !== 0 && y2 - y1 < toEle.height * 2; //如果结束点 开始点上下间距过小 第三个点y距离增加

            if (y1 === y2) {
                //&& !crossDomain
                points = [...points, x2, y2];
            } else {
                let inflectionPoint_1_x = 30 + x1,
                    inflectionPoint_1_y = y1,
                    inflectionPoint_2_x = inflectionPoint_1_x,
                    inflectionPoint_2_y = y2;
                breakPoints = [
                    inflectionPoint_1_x,
                    inflectionPoint_1_y,
                    inflectionPoint_2_x,
                    inflectionPoint_2_y
                ];

                points = [...points, ...breakPoints, x2, y2];
            }
            line.setAttribute("points", points.join(","));
            line.setAttribute("stroke", "#547a84");
            line.setAttribute("fill", "none");
            line.setAttribute(
                "id",
                (Math.random() * (36 - 1) + 1).toString(36).substr(2, 5)
            );
            line.setAttribute("stroke-width", 1);
            line.setAttribute("class", "graphLine");
            line.setAttribute("marker-end", "url(#arrowEnd)");
            return line;
        },

        /**
         * 绘制关系线文字描述
         */
        drawText(element, x1, x2, y1, y2, svg) {
            if (Object.prototype.hasOwnProperty.call(element, "text")) {
                let text = document.createElementNS(
                    "http://www.w3.org/2000/svg",
                    "text"
                );
                text.setAttribute(
                    "x",
                    (x2 - x1) / 2 + x1 - (element.text.length / 2) * 10
                );
                text.setAttribute("y", y1 + (y2 - y1) / 2 + 5);
                text.setAttribute("fill", "#ffffff");
                text.innerHTML = element.text;

                svg.appendChild(text);
            }
        },
        /**
         * 绘制关系模块
         */
        renderGraph() {
            // this.graphData
            let domFrageMent = document.createDocumentFragment();
            //泳道独立元素
            let dependenciesData = this.graphData.filter(
                v => v.parent == "-1" && v.lane !== 0
            ),
                //独立的第一泳道元素
                dependenciesData1 = this.graphData.filter(
                    (v, i, arr) => v.lane === 0 && arr.every(vv => vv.parent !== v.id)
                ),
                //关系元素
                renderDatas = this.graphData.filter(
                    (v, i, arr) =>
                        dependenciesData.every(vv => vv.id !== v.id) &&
                        dependenciesData1.every(vv => vv.id !== v.id)
                );

            for (let index = renderDatas.length - 1; index >= 0; index--) {
                //找到当前泳道对应的数据--最后一个泳道开始布局
                //  let index = this.titleData.length - 1;
                let laneData = renderDatas.filter(v => v.lane === index);
                if (laneData.length > 0) {
                    //找到父元素相同的
                    domFrageMent.appendChild(this.renderLaneByParnt(laneData, index));
                }
            }
            //绘制泳道独立元素
            let newArr = [...dependenciesData, ...dependenciesData1];
            for (let index = newArr.length - 1; index >= 0; index--) {
                domFrageMent.appendChild(
                    this.renderLaneByParnt([newArr[index]], newArr[index].lane)
                );
            }

            this.$refs.graph.appendChild(domFrageMent);
        },

        /**
         * 根据父id 分类渲染
         * @param {*} data 泳道数据
         *
         * @param {*} index 泳道下标
         */
        renderLaneByParnt(data, laneIndex) {
            //根据泳道父 id渲染
            let array = this.sortByParentId(data),
                domFrageMent = document.createDocumentFragment();

            for (let index = 0; index < array.length; index++) {
                const element = array[index];

                //计算当前同级元素已经绘制数量
                let beroreRender =
                    index === 0 ? 0 : array[Math.max(index - 1, 0)].length;
                //绘制高度 取决于最后一个泳道已经绘制元素数量
                let hasRednerCount = this.renderPosition.filter(
                    v => v.laneIndex === this.titleData.length - 1
                ).length;

                domFrageMent.appendChild(
                    this.renderLaneElement(element, laneIndex, hasRednerCount)
                );
            }
            return domFrageMent;
        },
        //根据泳道f父id 逐步绘制
        sortByParentId(data) {
            let arr = [];
            for (let index = 0; index < data.length; index++) {
                const element = data[index];
                let existData = arr.findIndex(v =>
                    v.some(vv => vv.parent === element.parent)
                );
                if (existData > -1) {
                    arr[existData] = [...arr[existData], element];
                } else {
                    arr.push([element]);
                }
            }
            return arr;
        },
        /**
         * 绘制泳道数据
         * @param {*} data 当前泳道数据
         * @param {*} laneIndex 泳道索引--查找泳道dom
         * @param {*} levelIndex 同泳道下已经绘制元素数量
         */
        renderLaneElement(data, laneIndex, levelIndex) {
            // 泳道位置信息
            let laneEle = this.$refs[
                "header_" + laneIndex
            ][0].getBoundingClientRect(),
                domFrageMent = document.createDocumentFragment();

            //  console.log(laneEle);
            for (let index = 0; index < data.length; index++) {
                //已经绘制的元素不在绘制
                if (this.renderPosition.find(v => v.id === data[index].id)) continue;
                let element = data[index],
                    id = element.id,
                    dataParent = element.parent;

                let top = this.getItemPositionTop(
                    element,
                    laneIndex,
                    index,
                    levelIndex
                );

                domFrageMent.appendChild(this.getItemElement(element, top));

                domFrageMent.appendChild(this.renderItemParent(element));
            }
            return domFrageMent;
        },
        /**
         * 模块高度
         * @param {*} element 模块data
         * @param {*} laneIndex 模块所在用到
         * @param {*} index 同级下标
         * @param {*} levelIndex 同泳道下已经绘制元素数量
         */
        getItemPositionTop(element, laneIndex, index, levelIndex) {
            console.log("求高度", element.name);
            let childPosition = this.getLaneChildrenElement(element.id, laneIndex),
                top = 0;
            if (
                element.lane === this.titleData.length - 1 &&
                this.renderPosition.length > 0
            ) {
                top =
                    Math.max(...this.renderPosition.map(v => v.top)) + this.itemHeight;
            } else {
                top =
                    childPosition === 0
                        ? this.getPanePosition(laneIndex, index, levelIndex)
                        : childPosition;
                //如果当前数据没有 关联
                if (element.parent == "-1" && childPosition === 0) {
                    //已经有连线关系的最大值
                    let dependencyData = this.renderPosition.filter(
                        v => v.relationshipElements === true
                    );
                    if (dependencyData.length > 0) {
                        top = Math.max(
                            Math.max(...dependencyData.map(v => v.top)) + this.itemHeight,
                            top
                        );
                    }
                }
            }

            return top;
        },
        /**
         * 获取当前泳道已经绘制及其子最大高度值
         */
        getMaxLaneHeight() { },
        //绘制当前父节点
        renderItemParent(element) {
            //绘制当前的父元素--并且当前的父元素子元素已经绘制完成
            let domFrageMent = document.createDocumentFragment();
            let parentData = this.graphData.find(v => v.id === element.parent);
            if (parentData) {
                //已经绘制过的子元素数量
                let renderChildCount = this.renderPosition.filter(
                    v => v.parent === parentData.id
                );
                //所有子元素数量
                let allChildCount = this.graphData.filter(
                    v => v.parent === parentData.id
                );
                //子元素全部绘制完 绘制当前
                if (allChildCount.length === renderChildCount.length) {
                    //当前泳道已经绘制元素数量
                    let hasRednerCount = this.renderPosition.filter(
                        v => v.lane === parentData.lane
                    );

                    domFrageMent.appendChild(
                        this.renderLaneElement(
                            [parentData],
                            parentData.lane,
                            hasRednerCount.length
                        )
                    );
                } else {
                    //绘制子元素
                    for (let index = 0; index < allChildCount.length; index++) {
                        //当前子 是否绘制完
                        domFrageMent.appendChild(
                            this.renderItemChildren(allChildCount[index])
                        );
                        domFrageMent.appendChild(
                            this.renderLaneByParnt(
                                [allChildCount[index]],
                                allChildCount[index].lane
                            )
                        );
                    }
                }
            }

            return domFrageMent;
        },
        renderItemChildren(element) {
            let domFrageMent = document.createDocumentFragment();
            //当前子元素
            let childrenData = this.graphData.filter(v => v.parent === element.id);
            //已经渲染子数量
            let hasRenderChildCount = this.renderPosition.filter(
                v => v.parent === element.id
            );
            if (hasRenderChildCount < childrenData) {
                //绘制子元素
                for (let index = 0; index < childrenData.length; index++) {
                    domFrageMent.appendChild(
                        this.createElementByItem(childrenData[index])
                    );
                }
                console.log(element.name, "渲染子级", childrenData);
            }
            return domFrageMent;
        },
        createElementByItem(element) {
            let domFrageMent = document.createDocumentFragment();
            //当前子级是否拥有子级
            domFrageMent.appendChild(this.renderItemChildren(element));

            //使用子级高度值或者一渲染元素最大高度值
            let top = element.childrens
                ? this.getLaneChildrenElement(element.id, element.lane)
                : Math.max(...this.renderPosition.map(v => v.top)) + this.itemHeight;
            domFrageMent.appendChild(this.getItemElement(element, top));

            return domFrageMent;
        },
        /**
         * 获取 item dom
         * @param element item  data
         * @param  top  定位元素top
         */
        getItemElement(element, top) {
            let ele = document.createElement("div"),
                laneEle = this.$refs[
                    "header_" + element.lane
                ][0].getBoundingClientRect();

            ele.innerHTML = element.name;
            ele.className = "graph-item item_primary ";
            ele.id = "item_" + element.id;
            ele.style = `position:absolute;top:${top}px;left:${element.lane *
                laneEle.width +
                laneEle.width / 2 -
                this.itemMinWidth / 2}px;width:${this.itemMinWidth
                }px;background-image:url(${this.images})`;

            //保存子元素的位置信息
            //relationshipElements 是否为独立元素
            this.renderPosition.push({
                id: element.id,
                parent: element.parent,
                top: top,
                laneIndex: element.lane,
                relationshipElements: element.parent != -1 && element.lane !== 0
            });
            const that = this;
            ele.addEventListener("click", function () {
                that.clickItem(element);
            });
            return ele;
        },
        /**
         * 绘制
         * @param {*} laneIndex 泳道
         * @param {*} index 当前 组下标
         * @param {*} levelIndex   同泳道下已经绘制元素数量
         */
        getPanePosition(laneIndex, index, levelIndex) {
            let minTop = this.itemHeight * index + 10; // this.itemHeight * (index + levelIndex) + 10
            //未绘制定位开始位置
            if (this.renderPosition.length === 0) return minTop;
            //当前泳道已经绘制元素
            let renderArr = this.renderPosition.filter(
                v => v.laneIndex === laneIndex
            ),
                renderItemData =
                    Math.max(...renderArr.map(v => v.top)) + this.itemHeight, //当前泳道已经绘制元素+ 下一个高度值
                //最后一个泳道已经绘制高度最大值
                lastLaneMax = Math.max(
                    ...this.renderPosition
                        .filter(
                            v =>
                                v.relationshipElements &&
                                v.laneIndex === this.titleData.length - 1
                        )
                        .map(v => v.top)
                );

            return Math.max(renderItemData, minTop, lastLaneMax); //+ this.itemHeight + 10
        },
        /**
         * 当前数据的子元素，定位父元素的位置
         * @param {*} id 当前绘制模块的id
         */
        getLaneChildrenElement(id, lane) {
            //父元素集合
            let laneChildren = this.renderPosition
                .filter(v => v.parent === id)
                .map(v => v.top);
            if (laneChildren.length === 0) return 0;
            else if (laneChildren.length === 1) return laneChildren[0];
            let firstChild = Math.min(...laneChildren),
                lastChild = Math.max(...laneChildren);
            return (lastChild - firstChild) / 2 + firstChild; // - 10//10 应该死模块的高度
        },
        getGraphData(graphData) {
            // //表头
            // this.titleData = [
            //   {
            //     name: "T0",
            //     id: 1,
            //     size: 1,
            //     desc: "T0时刻准备出发了"
            //   },
            //   {
            //     name: "T10",
            //     id: 2,
            //     size: 1,
            //     desc: "T10时刻 到达目的地"
            //   },
            //   {
            //     name: "T15",
            //     id: 3,
            //     size: 1,
            //     desc: "T15时刻 进攻前准备"
            //   },
            //   {
            //     name: "T20",
            //     id: 4,
            //     size: 1,
            //     desc: "T20时刻 发起进攻"
            //   },
            //   {
            //     name: "T30",
            //     id: 5,
            //     size: 1,
            //     desc: "T30时刻 攻占高地001"
            //   },
            //   {
            //     name: "T32",
            //     id: 6,
            //     size: 1,
            //     desc: "T10时刻 战役目的达成"
            //   }
            // ];
            // //关系数据
            // let data = [
            //   {
            //     name: "第一层级",
            //     lane: 0,
            //     id: "0",
            //     parent: -1,
            //     children: [
            //       {
            //         name: "泳道1",
            //         lane: 1,
            //         id: "1.1",
            //         parent: "0",
            //         children: [
            //           {
            //             name: "泳道2",
            //             lane: 2,
            //             id: "2.1",
            //             parent: "1.1",
            //             children: [
            //               {
            //                 name: "泳道3",
            //                 lane: 3,
            //                 id: "3.1",
            //                 parent: "2.1"
            //               },
            //               {
            //                 name: "泳道3---",
            //                 lane: 3,
            //                 id: "3.2",
            //                 parent: "2.1",
            //                 children: [
            //                   {
            //                     name: "泳道4--",
            //                     lane: 4,
            //                     id: "4.1",
            //                     parent: "3.2"
            //                   },
            //                   {
            //                     name: "泳道4",
            //                     lane: 4,
            //                     id: "4.2",
            //                     parent: "3.2",
            //                     children: [
            //                       {
            //                         name: "泳道5",
            //                         lane: 5,
            //                         id: "5.1",
            //                         parent: "4.2"
            //                       }
            //                     ]
            //                   }
            //                 ]
            //               },
            //               {
            //                 name: "泳道3",
            //                 lane: 3,
            //                 id: "3.3",
            //                 parent: "2.1",
            //                 children: [
            //                   {
            //                     name: "泳道5dd",
            //                     lane: 5,
            //                     id: "3.3.5.1",
            //                     parent: "3.3"
            //                   },
            //                   {
            //                     name: "泳道5dd",
            //                     lane: 5,
            //                     id: "3.3.5.2",
            //                     parent: "3.3"
            //                   },
            //                   {
            //                     name: "泳道5dd",
            //                     lane: 5,
            //                     id: "3.3.5.3",
            //                     parent: "3.3"
            //                   },
            //                   {
            //                     name: "泳道哈哈哈",
            //                     lane: 5,
            //                     id: "3.3.5.4",
            //                     parent: "3.3"
            //                   },
            //                   {
            //                     name: "泳道哈哈哈",
            //                     lane: 5,
            //                     id: "3.3.5.5",
            //                     parent: "3.3"
            //                   }
            //                 ]
            //               },
            //               {
            //                 name: "？？？",
            //                 lane: 3,
            //                 id: "3.5",
            //                 parent: "2.1",
            //                 children: [
            //                   // {
            //                   //   name: "这个呢1",
            //                   //   lane: 4,
            //                   //   id: "3.5.1dd",
            //                   //   parent: "3.5"
            //                   // },
            //                   {
            //                     name: "这个呢2",
            //                     lane: 4,
            //                     id: "3.5.2",
            //                     parent: "3.5",
            //                     children: [
            //                       {
            //                         name: "这个呢",
            //                         lane: 5,
            //                         id: "3.5.2.1",
            //                         parent: "3.5.2"
            //                       }
            //                     ]
            //                   }
            //                 ]
            //               },
            //               {
            //                 name: "泳道3",
            //                 lane: 3,
            //                 id: "3.4",
            //                 parent: "2.1",
            //                 children: [
            //                   {
            //                     name: "dfsdf",
            //                     lane: 4,
            //                     id: "3.4.5.2",
            //                     parent: "3.4"
            //                   },
            //                   {
            //                     name: "dfsdf",
            //                     lane: 4,
            //                     id: "3.4.5.1",
            //                     parent: "3.4",
            //                     children: [
            //                       {
            //                         name: "dfsdf",
            //                         lane: 5,
            //                         id: "3.4.5.1.1",
            //                         parent: "3.4.5.1"
            //                       }
            //                     ]
            //                   }
            //                 ]
            //               }
            //             ]
            //           },
            //           {
            //             name: "泳道2",
            //             lane: 2,
            //             id: "2.2",
            //             parent: "1.1",
            //             children: [
            //               {
            //                 name: "这个特殊的-1",
            //                 lane: 3,
            //                 id: "2.2.1",
            //                 parent: "2.2",
            //                 children: [
            //                   {
            //                     name: "这个特殊的-1",
            //                     lane: 4,
            //                     id: "2.2.1.1",
            //                     parent: "2.2.1"
            //                   },
            //                   {
            //                     name: "这个特殊的-2",
            //                     lane: 4,
            //                     id: "2.2.1.2",
            //                     parent: "2.2.1"
            //                   }
            //                 ]
            //               }
            //             ]
            //           }
            //         ]
            //       },
            //       {
            //         name: "泳道1",
            //         lane: 0,
            //         id: "1.2",
            //         parent: -1,
            //         children: [
            //           {
            //             name: "泳道2",
            //             lane: 1,
            //             id: "1.2.1",
            //             parent: "1.2",
            //             children: [
            //               {
            //                 name: "泳道3",
            //                 lane: 2,
            //                 id: "1.2.1.1",
            //                 parent: "1.2.1",
            //                 children: [
            //                   {
            //                     name: "泳道4",
            //                     lane: 3,
            //                     id: "1.2.1.1.1",
            //                     parent: "1.2.1.1",
            //                     children: [
            //                       {
            //                         name: "泳道4",
            //                         lane: 4,
            //                         id: "1.2.1.1.1.1",
            //                         parent: "1.2.1.1.1",
            //                         children: [
            //                           {
            //                             name: "最后一个用到吧",
            //                             lane: 5,
            //                             id: "aaa",
            //                             parent: "1.2.1.1.1.1"
            //                           },
            //                           {
            //                             name: "最后一个用到吧",
            //                             lane: 5,
            //                             id: "aaa1",
            //                             parent: "1.2.1.1.1.1"
            //                           }
            //                         ]
            //                       }
            //                     ]
            //                   },
            //                   {
            //                     name: "泳道4",
            //                     lane: 3,
            //                     id: "1.2.1.1.2",
            //                     parent: "1.2.1.1"
            //                   },
            //                   {
            //                     name: "泳道4",
            //                     lane: 3,
            //                     id: "1.2.1.1.3",
            //                     parent: "1.2.1.1"
            //                   },
            //                   {
            //                     name: "泳道4",
            //                     lane: 3,
            //                     id: "1.2.1.1.4",
            //                     parent: "1.2.1.1"
            //                   }
            //                 ]
            //               },
            //               {
            //                 name: "泳道3",
            //                 lane: 2,
            //                 id: "1.2.1.2",
            //                 parent: "1.2.1",
            //                 children: [
            //                   {
            //                     name: "泳道4",
            //                     lane: 3,
            //                     id: "1.2.1.2.1",
            //                     parent: "1.2.1.2"
            //                   }
            //                 ]
            //               },
            //               {
            //                 name: "泳道3",
            //                 lane: 2,
            //                 id: "1.2.1.3",
            //                 parent: "1.2.1"
            //               }
            //             ]
            //           }
            //         ]
            //       },
            //       {
            //         name: "三层",
            //         lane: 0,
            //         id: "a1",
            //         parent: -1
            //       }
            //     ]
            //   }
            // ];
            this.titleData = graphData.titleData;

            this.graphData = this.treeToArray(graphData.graphData);
            //  console.log(this.graphData);
            //关系数据
            this.dependenciesData = this.graphData.map(v => ({
                from: v.parent,
                to: v.id
            }));
            this.$nextTick(() => {
                this.itemMinWidth =
                    this.$refs.header_0[0].getBoundingClientRect().width - 60;
                this.renderGraph();
                this.praghHeight =
                    Math.max(...this.renderPosition.map(v => v.top)) + 20 + "px";
                //关系连线
                this.createLine();
            });
        },

        treeToArray(treeData) {
            var result = [];
            for (var key in treeData) {
                var obj = treeData[key];
                var clone = JSON.parse(JSON.stringify(obj));
                //添加自己数量 保证绘制顺序
                clone.childrens = clone.children ? clone.children.length : 0;
                delete clone.children;
                result.push(clone);
                //
                if (obj.children) {
                    var tmp = this.treeToArray(obj.children);
                    result = result.concat(tmp);
                }
            }
            return result;
        },
        /**
         * 点击事件
         * @param {*} data 当前数据
         */
        clickItem(data) {
            // this.imagesModal = true;
            this.$emit('click', data)
        },
        /**
         * 初始化数据
         *
         * @param {*} data
         */
        init(data) {
            //容器宽度
            let curELementRect = this.$el.getBoundingClientRect();
            this.conWidth = curELementRect.width;
            this.diffLeft = curELementRect.left;
            this.$nextTick(() => {
                this.getGraphData(data);
            });
        },
        /**
         *重置
         */
        resetGraph() {
            let allGraphEle = document.getElementById("graphMain");

            this.titleData = [];
            let polylineDom = document.querySelectorAll(".graphLine");
            //清空模块
            allGraphEle.innerHTML = "";
            this.praghHeight = 0;
            //移除连线
            for (let index = 0; index < polylineDom.length; index++) {
                polylineDom[index].remove();
            }
            //清空存储模块位置信息
            this.renderPosition = [];
        }
    },
    computed: {
        graphMainStyle() {
            // let maxLevel = this.data.map(v => Math.max(...v.map(vv => vv.length))).reduce((pre, a) => pre + a),
            //根据表头计算宽度
            let headerWidth =
                this.titleData.reduce((a, b) => {
                    return a + b.size;
                }, 0) * this.titleWidth;

            //表头宽度为当前容器与计算宽度最大值
            headerWidth = Math.max(headerWidth, this.conWidth);
            return {
                width: headerWidth + "px"
                //  height: maxLevel * this.itemHeight + this.headerHeight + "px" // 40为表头高度
            };
        }
    },
    created() { },
    mounted() {
        //初始化
        //  this.init();
        //重置
        // this.resetGraph();
    }
};
</script>

<style >
.graph-main {
    position: absolute;
    top: 50px;
    left: 0;
    width: 100%;
    height: calc(100vh - 180px);
    overflow: auto;
    background-color: #082634;
}
.graph-svg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}
.graph-main-container {
    width: 100%;
    height: 100%;
    position: relative;
    z-index: 10;
}
.graph-content {
    position: absolute;
    top: 100px;
    left: 0;
    right: 0;
    bottom: 4px;
    overflow: auto;
}
.graph-header {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100px;
    display: flex;
    align-items: center;
    justify-content: center;
}
.graph-header-item {
    height: 100%;
    min-width: 180px;
    /* padding: 9px 0;
  box-sizing: border-box; */
    border-right: 4px solid #092736;
    /* background-color: #0d3b4a; */
    font-size: 16px;
    font-weight: 400;
    color: #ccc;
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
}
.graph-header-item-title {
    height: 40px;
    line-height: 40px;
    width: 100%;
    background-color: #0d3b4a;
}
.desc {
    height: 60px;
    width: 100%;
    padding: 4px 10px;
    box-sizing: border-box;
    background-color: rgba(13, 59, 74, 0.56);
    font-size: 14px;
    text-align: center;
    overflow: auto;
}

.graph-content-level {
    width: 100%;
    margin-bottom: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    z-index: 10;
}
.graph-content-level > div {
    height: 100%;
}
.graph-content-level-item {
    display: flex;
    align-items: center;
    justify-content: space-around;
    flex-direction: column;
}
.level-item-group {
    display: flex;
    align-items: center;
    justify-content: space-around;
    flex-direction: row;
}
.level-item-group > div {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100%;
}
.graph-item {
    /* min-width: 100px;
  padding: 2px 15px;
  box-sizing: border-box;
  line-height: 28px;
  background-color: #127fee;
  font-size: 20px; */
    text-align: center;
    font-size: 14px;
    color: #ffffff;
    cursor: pointer;
}
.graph-content-level-item-group {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: space-around;
    flex-direction: column;
}
.graph-item-con {
    height: 100%;
    width: 100%;
}

.item_primary {
    height: 100px;
    width: 140px;
    /* padding: 2px 15px; */
    border-radius: 3px;
    box-sizing: border-box;
    line-height: 28px;
    /* background-color: #127fee; */
    background: url() no-repeat center center;
    background-size: auto 100%;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    border: 1px solid #127fee;
}
.item_circle {
    width: 85px;
    height: 85px;
    padding: 0;
    border-radius: 50%;
    line-height: 85px;
    background-color: #00a279;
    color: #ffffff;
}

.item_diamond {
    width: 110px;
    height: 40px;
    line-height: 30px;
    background: linear-gradient(135deg, transparent 10px, #6666cc 0) top left,
        linear-gradient(-135deg, transparent 10px, #6666cc 0) top right,
        linear-gradient(-45deg, transparent 10px, #6666cc 0) bottom right,
        linear-gradient(45deg, transparent 10px, #6666cc 0) bottom left;
    background-size: 50% 50%;
    background-repeat: no-repeat;
    font-size: 18px;
    color: #ffffff;
    line-height: 40px;
}
.special-dependency {
    pointer-events: visibleStroke;
    fill: transparent;
    marker-end: url(#arrowEnd);
    /* stroke: #bbb; */
    stroke-width: 1;
    transition: stroke-width 0.2s linear;
}
.graph-content-main {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    min-height: 100%;
}
.graph-main-items {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 100;
}
.graph-images {
    width: 100%;
}
</style>