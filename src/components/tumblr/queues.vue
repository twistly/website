<template>
    <div class="row">
        <div v-if="displayErrors" class="debug">{{newQueue}}</div>
        <div v-if="displayErrors" class="error">{{error}} {{stack}}</div>
        <div class="col-md-12 col-sm-12">
            <button @click="showModal" class="button">Create a new queue?</button>
            <modal name="new-queue" :adaptive="true">
                <div class="form-panel box" style="box-shadow: none;">
                    <form v-on:submit.prevent="checkQueue" class="form-horizontal clearfix">
                        <div class="form-group">
                            <label class="col-sm-4 control-label">Queue name</label>
                            <div class="col-sm-8">
                                <input v-model="newQueue.name" :placeholder="newQueue.blogs.length ? newQueue.blogs + ' queue' : ''" class="form-control"/>
                            </div>
                        </div>
                        <div class="form-group">
                            <label class="col-sm-4 control-label">Amount of posts per queue</label>
                            <div class="col-sm-8">
                                <input type="number" v-model="newQueue.interval" min="1" max="250" class="form-control"/>
                            </div>
                        </div>
                        <div class="form-group">
                            <label class="col-sm-4 control-label">Blogs that use this queue</label>
                            <div class="col-sm-8">
                                <input v-model="newQueue.blogs" :placeholder="blogs.map(blog => blog.url).join(', ')" class="form-control"></input>
                            </div>
                        </div>
                        <div class="form-group">
                            <label class="col-sm-4 control-label">Start time (0 - 23)</label>
                            <div class="col-sm-8">
                                <input type="number" v-model="newQueue.startTime" min="0" max="23" class="form-control"/>
                            </div>
                        </div>
                        <div class="form-group">
                            <label class="col-sm-4 control-label">End time (0 - 24)</label>
                            <div class="col-sm-8">
                                <input type="number" v-model="newQueue.endTime" min="0" max="24" class="form-control"/>
                            </div>
                        </div>
                        <button class="button">Add queue</button>
                    </form>
                </div>
            </modal>
        </div>
        <template v-if="queues.length >= 1">
            <div v-for="queue in queues" class="form-panel col-md-3 col-sm-3 mb box">
                <form class="form-horizontal">
                    <div class="form-group">
                        <label class="col-sm-8 control-label">Name</label>
                        <div class="col-sm-4">
                            <p class="form-control-static">{{queue.name}}</p>
                        </div>
                    </div>
                    <div class="form-group">
                        <label class="col-sm-8 control-label">Start time</label>
                        <div class="col-sm-4">
                            <p class="form-control-static">{{queue.startTime / ONE_HOUR}}</p>
                        </div>
                    </div>
                    <div class="form-group">
                        <label class="col-sm-8 control-label">End time</label>
                        <div class="col-sm-4">
                            <p class="form-control-static">{{queue.endTime / ONE_HOUR}}</p>
                        </div>
                    </div>
                    <div class="form-group">
                        <label class="col-sm-8 control-label">Amount of posts</label>
                        <div class="col-sm-4">
                            <p class="form-control-static">{{queue.interval}}</p>
                        </div>
                    </div>
                    <div class="form-group">
                        <label class="col-sm-8 control-label">Last run time</label>
                        <div class="col-sm-4">
                            <p class="form-control-static">
                                <time :datetime="queue.lastRun">{{queue.lastRun}}</time>
                            </p>
                        </div>
                    </div>
                    <div class="form-group">
                        <label class="col-sm-8 control-label">Next run time</label>
                        <div class="col-sm-4">
                            <p class="form-control-static">
                                <time :datetime="queue.lastRun">{{queue.nextRun}}</time>
                            </p>
                        </div>
                    </div>
                </form>
                <button @click="deleteQueue(queue._id)" class="button">Delete</button>
                <button @click="shuffleQueue(queue._id)" class="button">Shuffle</button>
            </div>
        </template>
        <div v-else>
            No Queues found.
        </div>
    </div>
</template>

<script>
import {mapGetters, mapActions} from 'vuex';

const ONE_SECOND = 1000;
const ONE_MINUTE = 60 * ONE_SECOND;
const ONE_HOUR = 60 * ONE_MINUTE;

export default {
    name: 'queues',
    props: {
        displayErrors: Boolean
    },
    data() {
        return {
            error: null,
            stack: null,
            loading: false,
            newQueue: {
                name: null,
                blogs: [],
                interval: null,
                startTime: null,
                endTime: null
            },
            ONE_SECOND,
            ONE_MINUTE,
            ONE_HOUR
        };
    },
    mounted() {
        const vm = this;
        vm.loading = true;
        vm.getQueues().catch(() => {
            vm.error = vm.queueError || null;
            vm.stack = vm.queueStack || null;
            vm.showError = vm.hasRole('admin') || false;
        }).then(vm.finishedLoading);
    },
    computed: {
        ...mapGetters([
            'blogs',
            'hasRole',
            'queues',
            'queueError',
            'queueStack',
            'isAuthenticated'
        ])
    },
    methods: {
        ...mapActions([
            'getQueues',
            'addQueue',
            'deleteQueue'
        ]),
        finishedLoading() {
            const vm = this;
            return new Promise(resolve => {
                vm.loading = false;
                resolve();
            });
        },
        showModal() {
            this.$modal.show('new-queue');
        },
        checkQueue() {
            const vm = this;
            const {name, blogs, interval, startTime, endTime} = vm.newQueue;
            const queue = Object.assign({}, {
                name,
                blogs,
                interval,
                startTime: startTime * ONE_HOUR,
                endTime: endTime * ONE_HOUR
            });

            if (blogs.length === 0) {
                this.error = `You need to add atleast one blog to the queue.`;
                return vm.finishedLoading();
            }

            this.$modal.hide('new-queue');
            vm.loading = true;

            queue.blogs = queue.blogs.split(', ').filter(x => x.trim()).map(blog => {
                const found = vm.blogs.filter(foundBlog => foundBlog.url === blog);
                return found.length ? found[0]._id : undefined;
            }).filter(x => x);

            if (queue.blogs.length === 0 && blogs.length >= 1) {
                this.error = `You don't have access to that blog.`;
                return vm.finishedLoading();
            }

            this.addQueue(queue).catch(error => {
                this.error = error;
            }).then(() => {
                return vm.finishedLoading();
            });
        }
    }
};
</script>

<style>
.form-horizontal.style-form .form-group:last-of-type {
    border: none;
    padding-bottom: 0;
    margin-bottom: 0;
}

.box {
    background-color: #2a2d35;
    margin: 10px;
    padding: 10px;
    box-shadow: 0px 1px 3px #202125;
    text-align: left;
}

.v--modal-box.v--modal {
    background: 0;
    box-shadow: none;
}

input.form-control {
    border-radius: 0;
    padding: 4px;
}

.clearfix::after {
   content: " ";
   display: block;
   height: 0;
   clear: both;
}
</style>
