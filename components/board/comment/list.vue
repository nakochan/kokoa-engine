<template>
    <div
        ref='commentList'
        v-shortkey.once='["c"]'
        @shortkey='scrollToCommentList'>
        <b-modal
            id='bv-comment-remove-modal'
            @ok='remove'
            title='알림'
            title-tag='h6'
            cancel-title='취소'
            ok-title='확인'
            size='sm'
            auto-focus-button='ok'
            centered>
            정말로 삭제하시겠습니까?
        </b-modal>
        <article class='comment-view'>
            <h6>댓글 <span>[{{ numberWithCommas(postsCount) }}]</span></h6>
            <ul v-if='postsCount > 0'>
                <li
                    :ref='`post${item.id}`'
                    v-for='(item, index) in posts' :key='index'>
                    <div>
                        <div class='re' v-if='item.tagUserId'/>
                        <div class='image' v-if='item.profile'>
                            <img :src='`/profile/${item.profile}`' @error='imageUrlAlt'>
                        </div>
                        <div class='image' v-else>
                            <div>
                                <client-only>
                                    <avatar
                                        :username='item.author'
                                        :inline='true'
                                        :size='48'
                                        :rounded='false'/>
                                </client-only>
                            </div>
                        </div>
                        <div class='content'>
                            <div>
                                <img :src='`/level/${item.level}.png`'>
                                <img class='icon' :src='`/icon/${item.icon}`' v-if='item.icon !== ""'>
                                <span class='author'>{{ item.author }}</span>
                                <span class='regdate'>{{ $moment(item.updated).fromNow() }}</span>
                            </div>
                            <div :class='item.userId === topic.userId ? "writer" : ""'>
                                <div class='tagUser' v-if='item.tagUserId'>
                                    <font-awesome-icon icon='at'/>
                                    {{ item.tagAuthor }}
                                </div>
                                <div class='sticker' v-if='item.stickerId > 0'>
                                    <img :src='`/sticker/${item.stickerId}/${item.stickerSelect}`'>
                                </div>
                                <span v-html='item.content'/>
                            </div>
                        </div>
                        <div class='more'>
                            <b-dropdown size='sm' right no-caret>
                                <b-dropdown-item @click='handleCommand(["reply", item.id])'>대댓글</b-dropdown-item>
                                <b-dropdown-item @click='handleCommand(["votes", item.id, true])' v-if='item.userId !== $store.state.user.id'>추천</b-dropdown-item>
                                <b-dropdown-item @click='handleCommand(["votes", item.id, false])' v-if='item.userId !== $store.state.user.id'>비추천</b-dropdown-item>
                                <b-dropdown-item @click='handleCommand(["update", item])' v-if='$store.state.user.isAdmin > 0 || item.userId === $store.state.user.id'>수정</b-dropdown-item>
                                <b-dropdown-item @click='handleCommand(["remove", item.id])' v-if='$store.state.user.isAdmin > 0 || item.userId === $store.state.user.id'>삭제</b-dropdown-item>
                            </b-dropdown>
                            <div>
                                <font-awesome-icon icon='ellipsis-h'/>
                            </div>
                        </div>
                    </div>
                    <div v-if='item.id === tempPostReplyId'>
                        <PostWrite
                            :id='id'
                            :pureContent='""'
                            :author='item.author'
                            :topicUserId='topic.userId'
                            :postUserId='item.userId'
                            :postRootId='item.postRootId || item.id'
                            :postParentId='item.id'/>
                    </div>
                    <div v-if='item.id === tempPostUpdateId'>
                        <PostWrite
                            :id='item.id'
                            :edit='true'
                            :pureContent='item.content.replace(/<br>+/g, "\n")'/>
                    </div>
                </li>
            </ul>
            <div class='postBox'>
                <PostWrite
                    :id='id'
                    :pureContent='""'
                    :topicUserId='topic.userId'/>
            </div>
        </article>
        <b-button-group class='mt-3' v-if='newPostsCount > 0'>
            <b-button
                variant='primary'
                @click='getData'
                pill>
                <font-awesome-icon class='fa-spin' icon='sync-alt'/>
                새 댓글 불러오기 ({{ numberWithCommas(newPostsCount) }})
            </b-button>
        </b-button-group>
    </div>
</template>

<script>
    import PostWrite from '~/components/board/comment/write.vue'
    
    export default {
        components: { PostWrite },
        props: ['id', 'topic'],
        data() {
            return {
                posts: [],
                postsCount: 0,
                postsPage: 1,
                newPostsCount: 0,
                viewPostId: 0,
                tempPostReplyId: 0,
                tempPostUpdateId: 0,
                tempPostDeleteId: 0,
                loading: false
            }
        },
        watch: {
            '$store.state.forceUpdate': function() {
                this.getData()
            }
        },
        beforeMount() {
            this.$socket.on('newPost', () => {
                this.playSound('https://soundbible.com/mp3/Blop-Mark_DiAngelo-79054334.mp3')
                this.newPostsCount++
            })
            this.$socket.on('votePost', data => {
                this.posts = this.posts.map(post => {
                    if (post.id === data.postId) {
                        post.likes = data.likes
                        post.hates = data.hates
                    }
                    return post
                })
            })
        },
        mounted() {
            this.viewPostId = this.$route.query.postId
            this.getData()
        },
        methods: {
            getData: async function() {
                if (this.loading)
                    return
                this.loading = true
                const data = await this.$axios.$post('/api/topic/list/post', { id: this.id, page: this.postsPage - 1 })
                if (data.status === 'fail') {
                    this.loading = false
                    return this.toast(data.message || '오류가 발생했습니다.', 'danger')
                }
                this.postsCount = data.count
                this.newPostsCount = 0
                this.tempPostReplyId = 0
                this.tempPostUpdateId = 0
                const regex1 = /([--:\w?@%&+~#=]*\.[a-z]{2,4}\/{0,2})((?:[?&](?:\w+)=(?:\w+))+|[--:\w?@%&+~#=]+)?/g
                const regex2 = /\[\[\s*hy(\d+)\]\]/gi
                const regex3 = /\[\[\s*ht(\d+)\]\]/gi
                const regex4 = /\[\[\s*av([a-zA-Z0-9\_-]+(\d+))\]\]/gi
                if (data.posts) {
                    this.posts = data.posts.map(item => {
                        if(item.content.search(/((\.\.\.))/g) == -1) {
                            if(item.content.search(/((http(s)?)(:\/\/)?)/g) == -1) {
                                item.content = item.content.replace(regex1, '<a href="https://$&" target="_blank">$&</a>') 
                            } else {
                                item.content = item.content.replace(regex1, '<a href="$&" target="_blank">$&</a>')   
                            }
                        } 
                        item.content = item.content.replace(regex2, '<a href="https://hiyobi.me/reader/$1" target="_blank" class="hitomiReader">hiyobi:$1</a>') 
                        item.content = item.content.replace(regex3, '<a href="https://hitomi.la/reader/$1.html" target="_blank" class="hitomiReader">hitomi:$1</a>')     
                        item.content = item.content.replace(regex4, '<a href="https://www.avdbs.com/menu/dvd_list.php?_kwd=$1" target="_blank" class="hitomiReader">av:$1</a>') 
                        return item
                    })
                }
                if (this.viewPostId > 0)
                    this.scrollTo()
                this.loading = false
            },
            handleCommand(command) {
                switch (command[0]) {
                case 'reply':
                    this.reply(command[1])
                    break
                case 'votes':
                    this.votes(command[1], command[2])
                    break
                case 'update':
                    this.update(command[1])
                    break
                case 'remove':
                    this.removeHandler(command[1])
                    break
                }
            },
            reply(id) {
                this.tempPostReplyId = id
                this.tempPostUpdateId = 0
            },
            votes: async function(id, flag) {
                if (id < 1)
                    return
                if (!this.$store.state.user.isLogged)
                    return this.toast('로그인하세요.', 'warning')
                const token = this.$store.state.user.token
                this.$store.commit('setLoading', true)
                const data = await this.$axios.$post(
                    '/api/topic/vote/post',
                    { id, likes: flag },
                    { headers: { 'x-access-token': token } }
                )
                if (data.status === 'fail') {
                    this.$store.commit('setLoading')
                    return this.toast(data.message || '오류가 발생했습니다.', 'danger')
                }
                this.toast('투표했습니다.', 'success')
                this.$store.commit('setLoading')
            },
            update(item) {
                if (item.id < 1)
                    return
                if (!this.$store.state.user.isLogged)
                    return this.toast('로그인하세요.', 'warning')
                this.tempPostReplyId = 0
                this.tempPostUpdateId = item.id
            },
            removeHandler: async function(id) {
                if (id < 1)
                    return
                this.tempPostDeleteId = id
                this.$bvModal.show('bv-comment-remove-modal')
            },
            remove: async function() {
                if (!this.$store.state.user.isLogged)
                    return this.toast('로그인하세요.', 'warning')
                const token = this.$store.state.user.token
                this.$store.commit('setLoading', true)
                const data = await this.$axios.$delete(
                    '/api/topic/delete/post',
                    {
                        data: { id: this.tempPostDeleteId, page: this.postsPage },
                        headers: { 'x-access-token': token }
                    }
                )
                if (data.status === 'fail')
                    return this.toast(data.message || '오류가 발생했습니다.', 'danger')
                this.posts = this.posts.filter(post => post.id !== this.tempPostDeleteId)
                --this.postsCount
                this.toast('댓글 삭제 성공!', 'success')
                this.$store.commit('setLoading')
            },
            currentChange(page) {
                this.postsPage = page
                this.getData()
            },
            scrollTo() {
                const el = this.posts.find(p => p.id == this.viewPostId)
                if (!el)
                    return
                this.$nextTick(() => {
                    window.scrollTo({
                        top: this.$refs[`post${this.viewPostId}`][0].offsetTop,
                        behavior: 'smooth'
                    })
                })
            },
            scrollToCommentList() {
                this.$nextTick(() => {
                    window.scrollTo({
                        top: this.$refs.commentList.offsetTop,
                        behavior: 'smooth'
                    })
                })
            },
            numberWithCommas(x) {
                return x.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ',')
            },
            imageUrlAlt(event) {
                event.target.src = 'https://github.com/u3u.png'
            },
            playSound(sound) {
                if (!sound)
                    return
                const audio = new Audio(sound)
                audio.play()
            },
            toast(text, variant = 'default') {
                this.$bvToast.toast(text, {
                    title: '알림',
                    toaster: 'b-toaster-top-center',
                    variant: variant,
                    solid: true,
                    appendToast: true
                })
            }
        }
    }
</script>

<style lang='less' scope>
    @primary: #9BA9FB;

    article.comment-view {
        > h6 {
            height: 32px;
            margin: 0;
            padding: .5rem 0 0 5px;
            color: #fff;
            font-size: 14px;
            border-bottom: 1px solid rgba(0, 0, 0, .1);
            border-top-left-radius: 10px;
            border-top-right-radius: 10px;
            background-color: @primary;
            > span { font-size: 12px }
        }
        > ul {
            margin: 0;
            padding: 0;
            list-style: none;
            > li {
                border-bottom: 1px solid #eee;
                background-color: #fff;
                &:last-child { border: 0 }
                > div:first-child {
                    display: flex;
                    margin: 0;
                    > .re {
                        padding-left: 32px;
                        background: url(data:image/gif;base64,R0lGODlhDAANAIABAKioqP///yH5BAEAAAEALAAAAAAMAA0AAAIZRI4ZpsqNHlwz1iMthoBl3C3dSIaceV5PAQA7) no-repeat;
                        background-position: 20px 22px;
                    }
                    > .image {
                        > img, > div {
                            width: 56px;
                            height: auto;
                            margin: 12px;
                            margin-right: 0;
                            padding: 3px;
                            background-color: #fff;
                            border: 1px solid #ddd;
                            > div { padding: 3px }
                        }
                    }
                    > .content {
                        flex: 1;
                        > div {
                            margin: 8px;
                            font-size: 12px;
                            word-break: break-all;
                            &:nth-child(1) {
                                > img:nth-child(1) { margin-top: -3px }
                                > img:nth-child(2) {
                                    width: 16px;
                                    height: 16px;
                                    margin-top: -3px;
                                    border-radius: 2px;
                                }
                                > span.author {
                                    color: rgb(68, 68, 68);
                                    font-weight: 700;
                                }
                                > span.regdate {
                                    padding: 0 6px;
                                    color: rgb(136, 136, 136);
                                    font-size: 11px;
                                }
                            }
                            &:nth-child(2) {
                                >.tagUser {
                                    display: inline-block;
                                    margin-right: .25rem;
                                    padding: 2px 6px 1px 6px;
                                    color: #fff;
                                    font-size: 11px;
                                    border-radius: 500rem;
                                    background-color: @primary;
                                }
                                > .sticker {
                                    width: 100px;
                                    height: 100px;
                                    margin-top: 5px;
                                    > img { border-radius: 4px }
                                }
                                > span {
                                    line-height: 1.5;
                                    color: #000;
                                    overflow-wrap: break-word;
                                }
                            }
                            &.writer > span {
                                color: #061EA4;
                                font-weight: 700;
                            }
                        }
                    }
                    > .more {
                        position: relative;
                        width: 2rem;
                        background: #f7f8fa;
                        > div {
                            position: absolute;
                            width: 100%;
                            height: 100%;
                            &.dropdown {
                                z-index: 1;
                                .btn-secondary {
                                    opacity: 0;
                                    &:focus, &.focus { box-shadow: none !important }
                                }
                            }
                            > svg {
                                position: absolute;
                                top: calc(50% - 8px);
                                margin-left: 8px;
                            }
                        }
                    }
                }
            }
        }
    }

    .dropdown-item {
        padding: 0.25rem .75rem;
        font-size: 13px;
    }
</style>