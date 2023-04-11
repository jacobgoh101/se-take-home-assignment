<script >
export default {
  data() {
    return {
      nextOrderId: 1,
      nextBotId: 1,
      /* {
  [id]: {
    id: number;
    createdAt: Date;
    type: 'regular' | 'vip';
    processedAt?: Date;
    processedBy?: number; // bot ID
    completedAt?: Date;
    completedBy?: number; // bot ID
  }
} */
      orders: {},
      /* {
  [id]: {
    id: number;
    createdAt: Date;
    destroyedAt?: Date;
    processingOrderId?: number;
    processingTimeoutId?: number; // an ID returned by `setTimeout`, may be used for `clearTimeout`
  }
} */
      bots: {},
      vipOrderQueue: [],
      regularOrderQueue: []
    }
  },
  computed: {
    orderQueue() {
      return [
        ...this.vipOrderQueue, ...this.regularOrderQueue
      ]
    },
    availableBots() {
      return Object.values(this.bots).filter(bot => !bot.processingOrderId && !bot.destroyedAt)
    },
    ordersInProgress() {
      return Object.values(this.bots).filter(bot => bot.processingOrderId && !bot.destroyedAt).map(
        bot => bot.processingOrderId
      ).map(
        orderId => this.orders[orderId]
      )
    },
    completedOrders() {
      return Object.values(this.orders).filter(order => order.completedAt)
    },
    shouldPickupOrder() {
      return this.availableBots.length > 0 && this.orderQueue.length > 0 ? `${this.availableBots[0].id} ${this.orderQueue[0].id}` : false;
    }
  },
  methods: {
    enqueue(order) {
      if (order.type === 'vip') {
        this.vipOrderQueue.push(order)
      } else if (order.type === 'regular') {
        this.regularOrderQueue.push(order)
      } else {
        throw new Error('Unknown order type')
      }
    },
    dequeue() {
      const order = this.orderQueue[0]
      if (!order) return null
      if (order.type === 'vip') {
        this.vipOrderQueue = this.vipOrderQueue.filter(iOrder => iOrder.id !== order.id)
      } else if (order.type === 'regular') {
        this.regularOrderQueue = this.regularOrderQueue.filter(iOrder => iOrder.id !== order.id)
      } else {
        throw new Error('Unknown order type')
      }
      return order;
    },
    addOrder(type) {
      if (type !== 'vip' && type !== 'regular') throw new Error('Unknown order type')
      const order = {
        id: this.nextOrderId++,
        createdAt: new Date(),
        type
      }
      this.orders[order.id] = order
      this.orders = { ...this.orders }
      this.enqueue(order)
      return order;
    },
    addBot() {
      const bot = {
        id: this.nextBotId++,
        createdAt: new Date()
      }
      this.bots[bot.id] = bot
      this.bots = { ...this.bots }
      return bot;
    },
    removeBot() {
      const bot = Object.values(this.bots)
        .filter(
          b => !b.destroyedAt
        )
        .sort(
          (a, b) => a.createdAt - b.createdAt
        ).pop()
      if (!bot) throw new Error('Bot not found');
      const id = bot.id
      this.bots[id].destroyedAt = new Date();
      if (bot.processingTimeoutId) {
        clearTimeout(bot.processingTimeoutId)
        this.bots[id].processingTimeoutId = undefined
      }
      if (bot.processingOrderId) {
        this.revertOrder(bot.processingOrderId)
        this.bots[id].processingOrderId = undefined
      }
      this.bots[id] = { ...this.bots[id] };
      this.bots = { ...this.bots };
    },
    revertOrder(orderId) {
      let order = this.orders[orderId];
      if (!order) throw new Error('Order not found');
      this.orders[orderId].processedAt = undefined;
      this.orders[orderId].processedBy = undefined;
      this.orders[orderId] = { ...this.orders[orderId] };
      this.orders = { ...this.orders };
      order = this.orders[orderId];
      if (order.type === 'vip') {
        const targetIndex = this.vipOrderQueue.map(
          (iOrder, iOrderIndex) => {
            if (order.createdAt < iOrder.createdAt) {
              return iOrderIndex;
            }
            return false;
          }
        ).filter(i => !isNaN(i))
        if (targetIndex) {
          this.vipOrderQueue.splice(targetIndex, 0, order)
        } else {
          this.vipOrderQueue.push(order)
        }
      } else if (order.type === 'regular') {
        const targetIndex = this.regularOrderQueue.map(
          (iOrder, iOrderIndex) => {
            if (order.createdAt < iOrder.createdAt) {
              return iOrderIndex;
            }
            return false;
          }
        ).filter(i => !isNaN(i))
        if (targetIndex) {
          this.regularOrderQueue.splice(targetIndex, 0, order)
        } else {
          this.regularOrderQueue.push(order)
        }
      } else {
        throw new Error('Unknown order type')
      }
    }
  },
  watch: {
    shouldPickupOrder(shouldPickupOrder) {
      if (!shouldPickupOrder) return;
      const order = this.dequeue()
      const availableBot = this.availableBots[0]
      this.bots[availableBot.id].processingOrderId = order.id
      this.orders[order.id].processedAt = new Date()
      this.orders[order.id].processedBy = availableBot.id
      const timeoutId = setTimeout(() => {
        if (this.bots[availableBot.id].destroyedAt) {
          console.error('Bot destroyed. Order processing cancelled.')
          return;
        }
        if (this.bots[availableBot.id].processingOrderId !== order.id) {
          console.error('mismatched order id. Order processing cancelled.')
          return;
        }
        if (this.orders[order.id].completedAt) {
          console.error('Order already completed. Order processing cancelled.')
          return;
        }
        this.orders[order.id].completedAt = new Date()
        this.orders[order.id].completedBy = availableBot.id
        this.bots[availableBot.id].processingOrderId = undefined
        this.bots[availableBot.id].processingTimeoutId = undefined


        this.bots[availableBot.id] = { ...this.bots[availableBot.id] }
        this.bots = { ...this.bots }
        this.orders[order.id] = { ...this.orders[order.id] }
        this.orders = { ...this.orders }
      }, 10000);
      this.bots[availableBot.id].processingTimeoutId = timeoutId

      this.bots[availableBot.id] = { ...this.bots[availableBot.id] }
      this.bots = { ...this.bots }
      this.orders[order.id] = { ...this.orders[order.id] }
      this.orders = { ...this.orders }

    }
  }
}
</script>

<template>
  <div>
    <button @click="addOrder('regular')">
      New Normal Order
    </button>
    <br>
    <button @click="addOrder('vip')">
      New VIP Order
    </button>
    <br>
    <button @click="addBot">
      + Bot
    </button>
    <button @click="removeBot">
      - Bot
    </button>
  </div>
  <br>
  <br>

  <div>
    Pending Order :
    <ul>
      <li v-for="order in orderQueue"
          :key="order.id">
        {{ order.id }} - {{ order.type }} - {{ order.createdAt }}
      </li>
    </ul>
  </div>
  <br>
  <br>
  <div>
    Order In Progress:
    <ul>
      <li v-for="order in ordersInProgress"
          :key="order.id">
        {{ order.id }} - {{ order.type }} - {{ order.createdAt }}
      </li>
    </ul>
  </div>
  <br>
  <br>
  <div>
    Completed Orders:
    <ul>
      <li v-for="order in completedOrders"
          :key="order.id">
        {{ order.id }} - {{ order.type }} - {{ order.createdAt }}
      </li>
    </ul>
  </div>
  <br><br>
  <div>
    bots:
    <ul>
      <li v-for="bot in bots"
          :key="bot.id">
        {{ bot.id }} - {{ bot.createdAt }} - {{ bot.destroyedAt }}
      </li>
    </ul>
  </div>
</template>

<style scoped>
</style>
