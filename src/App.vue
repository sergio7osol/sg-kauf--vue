<template>
  <header class="main-header"> <!-- navbar navbar-dark sticky-top bg-dark flex-md-nowrap p-0 shadow -->
  <a class="navbar-brand col-md-3 col-lg-2 me-0 px-3" href="/">
    <svg class="vue-logo" version="1.1" viewBox="0 0 261.76 226.69" xmlns="http://www.w3.org/2000/svg"><g transform="matrix(1.3333 0 0 -1.3333 -76.311 313.34)"><g transform="translate(178.06 235.01)"><path d="m0 0-22.669-39.264-22.669 39.264h-75.491l98.16-170.02 98.16 170.02z" fill="#41b883"/></g><g transform="translate(178.06 235.01)"><path d="m0 0-22.669-39.264-22.669 39.264h-36.227l58.896-102.01 58.896 102.01z" fill="#34495e"/></g></g></svg>
    SG Kauf
  </a>
    <button class="navbar-toggler position-absolute d-md-none collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#sidebarMenu" aria-controls="sidebarMenu" aria-expanded="false" aria-label="Toggle navigation">
      <span class="navbar-toggler-icon"></span>
    </button>
    <input class="form-control form-control-dark w-100" type="text" placeholder="Search" aria-label="Search">
    <ul class="navbar-nav px-3">
      <li class="nav-item text-nowrap">
        <a class="nav-link" href="#">Sign out</a>
      </li>
    </ul>
  </header>
  <div class="container-fluid main-wrapper">
    <div class="row">
      <div class="col main-content">
        <div id="spend-track">
          <div class="row">
            <div class="main-content__left-menu">
              <left-menu :dates='dates' @date-selected='getDate' :key="Date.now()" /> 
            </div>
            <div class="main-content__body col">
              <buy-list :dateBuys="activeDateBuys" @transfer-save-product="saveProduct" />
            </div>
          </div>
        </div>
      </div>
      <div class="col-2 y-auto aside">
        <sum-calc :amount="calculatedSum" :currency="activeCurrency" @get-calc-sum="getCalcSum" @get-whole-sum="getWholeSum" /> <!-- TODO: currency exchange      |:dateRange=""|         --> 
        <sum :date="activeDate" :amount="activeSum" :currency="activeCurrency" />
      </div>
    </div>
  </div> 
</template> 

<script>
import '../node_modules/normalize.css/normalize.css';
import '../node_modules/bootstrap/dist/css/bootstrap.min.css';
import LeftMenu from './components/LeftMenu.vue';
import BuyList from './components/BuyList.vue';
import Sum from './components/Sum';
import SumCalc from './components/SumCalc';

export default {
  name: 'App',
  data() {
    return {
      title: 'SG-Kauf--Vue',
      calculatedSum: 0,
      dates: [],
      activeDateBuys: [],
      start: null
    };
  },
  computed: {
    activeDate() {
      try {
        return this.activeDateBuys[0].date;
      } catch (error) {
        return 'No date selected';
      }
    },
    activeSum() {
      const sum = this.activeDateBuys.reduce((buySum, buy) => {
        const products = buy.products; 
        let resultProductSum = null;

        if (products) {
          resultProductSum = products.reduce((productSum, product) => {
            const { price, weightAmount, discount } = product;
            let lastLetter = null;
            let discountNumber = null;
            let discountFactor = null;

            // calculating cost
            productSum.cost += price * weightAmount

            if (typeof discount === 'string') {
              lastLetter = discount.slice(-1);

              if (lastLetter !== '%') {
                throw Error('The last symbol in the discount string value should be %. Program exits.');
              }

              discountNumber = Number(discount.slice(0, -1))
              discountFactor = (price/100) * discountNumber;

            } else if (typeof discount === 'number') {
              discountFactor = (price * discount/100); 
            } else {
              throw Error('"discount" product prop should be eigher persentage of type "string" ("%" at the end) or "number". Program exits.');
            }

            // calculating discount
            productSum.discount += discountFactor * weightAmount;

            return productSum;
          }, {cost: 0, discount: 0});

          buySum.cost += resultProductSum.cost;
          buySum.discount += resultProductSum.discount;
        }

        return buySum;
      }, {cost: 0, discount: 0});

      console.log('sum: ', sum);

      return sum;
    },
    activeCurrency() { // TODO: make converting to one currency for all buys (in case they are different)
      try {
        console.log('this.activeDateBuys[0]: ', this.activeDateBuys[0]);
        return this.activeDateBuys[0].currency;
      } catch (error) {
        return '';
      }
    }
  },
  methods: {
    getAllDates: function() {
      let dates = this.dates;

      fetch('http://localhost:3030/list-dates')
        .then(
          function(response) {
            if (response.status !== 200) {
              console.log('Looks like there was a problem. Status Code: ' + response.status);
              return;
            }

            response.json().then(function(data) {
              console.log('dates: ', data.length);
              data.forEach(date => {
                dates.push(date);
              });
            });
          }
        )
        .catch(function(err) {
          console.log('Fetch Error :-S', err);
        });
    },
    getDate: function(e) {
      let thisApp = this;
      let newDate = e;

      const dateToSelect = thisApp.dates.find(item => item.date === newDate);

      if (dateToSelect.buys) {
        this.activeDateBuys = dateToSelect.buys.slice();
        return true;
      }

      fetch(`http://localhost:3030/read-date?date=${newDate}`)
        .then(
          function(response) {
            if (response.status !== 200) {
              console.log('Looks like there was a problem. Status Code: ' + response.status);
              return;
            }

            response.json().then(function(data) {
              dateToSelect.buys = data;
              thisApp.activeDateBuys = dateToSelect.buys;
            });
          }
        )
        .catch(function(err) {
          console.log('Fetch Error :-S', err);
        });
    },
    addProductToDates: function () {
      this.getAllDates();
    },
    getCalcSum(range) {
      const thisApp = this;

      console.log('RANGE send: ', range);

      fetch(`http://localhost:3030/get-calc-sum?from=${range.from}&to=${range.to}`)
        .then(
          function(response) {
            if (response.status !== 200) {
              console.log('Looks like there was a problem. Status Code: ' + response.status);
              return;
            }

            response.json().then(function(data) {
              if (typeof data === 'object' && typeof data.cost === 'number' && typeof data.discount === 'number') {
                thisApp.calculatedSum = data;
              }
            });
          }
        )
        .catch(function(err) {
          console.log('Fetch Error :-S', err);
        });
    },
    getWholeSum() {
      const thisApp = this;

      fetch('http://localhost:3030/get-whole-sum')
        .then(
          function(response) {
            if (response.status !== 200) {
              console.log('Looks like there was a problem. Status Code: ' + response.status);
              return;
            }

            response.json().then(function(data) {
              const wholeSum = data.wholeSum;

              if (wholeSum && typeof wholeSum === 'object' && typeof wholeSum.cost === 'number' && typeof wholeSum.discount === 'number') {
                thisApp.calculatedSum = wholeSum;
              }
            });
          }
        )
        .catch(function(err) {
          console.log('Fetch Error :-S', err);
        });
    },
    saveProduct(productInfoForSave) {
      console.log('App -> productInfoForSave: ', productInfoForSave);
        const thisApp = this;
        const date = productInfoForSave.date;
        const time = productInfoForSave.time;
        let { name, price, weightAmount, measure, description, discount } = productInfoForSave.product;
        let url = `http://localhost:3030/save-product?date=${date}&time=${time}`;

        name = encodeURIComponent(name);

        url += name ? `&name=${name}` : '';
        url += price ? `&price=${price}` : '';
        url += weightAmount ? `&weightAmount=${weightAmount}` : '';
        url += measure ? `&measure=${measure}` : '';
        url += description ? `&description=${description}` : '';
        url += discount ? `&discount=${discount}` : '';

        fetch(url)
            .then((response) => {
                if (response.status !== 200) {
                    console.log('Looks like there was a problem. Status Code: ' + response.status);
                    return;
                }

                response.json().then(function (data) {
                    if (response.status !== 200) {
                        console.log('Error. Program stops. ', data.error);
                        return false;
                    } else {
                      const dateToAddProductTo = thisApp.dates && thisApp.dates.find(buyDate => buyDate.date === date);
                      if (!dateToAddProductTo) {
                        console.log(`Date ${date} to add the product to - is not found`);
                        return false;
                      }

                      const buyToAddProductTo = dateToAddProductTo && dateToAddProductTo.buys && dateToAddProductTo.buys.find(buy => buy.time === time);
                      if (!buyToAddProductTo) {
                        console.log(`Buy at ${time} to add the product to - is not found`);
                        return false;
                      }

                      buyToAddProductTo.products = data;

                      // react to changed products if same date is active
                      thisApp.displayNewProductState(thisApp.activeDateBuys, buyToAddProductTo, dateToAddProductTo);
                    }
                });
            })
            .catch(function (err) {
                console.log('Fetch Error :-S', err);
            });
    },
    countDateProducts() {
      return this.activeDateBuys.reduce((acc, v) => acc + v.products.length, 0);
    },
    // utils
    displayNewProductState(activeDateBuys, buyWithAddedProduct, dateToAddProductTo) {
      const activeDate = activeDateBuys && activeDateBuys[0] && activeDateBuys[0].date;
      const changedBuyDate = buyWithAddedProduct.date;
      let activeProductAmount = null;

      if (activeDate !== changedBuyDate) { 
        console.log(`Date ${changedBuyDate} which was added the product is not active anymore.`);
        return false;
      }

      const activeDateBuyToUpdate = activeDateBuys.find(buy => buy.time === buyWithAddedProduct.time);
      if (!activeDateBuyToUpdate) {
        console.log(`No buy at ${buyWithAddedProduct.time} for the active date ${activeDate} was found.`);
        return false;
      }

      activeDateBuyToUpdate.products = buyWithAddedProduct.products;
      activeProductAmount = this.countDateProducts();
      // save the new amount of products for a particular date: 
      dateToAddProductTo.count = activeProductAmount;
    }
  },
  created: function () {
    this.getAllDates();
  },
  components: {
    LeftMenu,
    BuyList,
    Sum,
    SumCalc
  },
};
</script>

<style lang="scss">
.vue-logo {
  width: 2rem;
}

.main-header {
  padding: 0;
  box-shadow: 0 .5rem 1rem rgba(0,0,0,.15)!important;
  background-color: #343a40!important;
  position: fixed;
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  z-index: 3;

  .navbar-brand, .nav-link {
    color: #fff;
  }
}

.main-content {
  &__left-menu {
    height: 100vh;
    overflow-y: auto;
    overflow-x: hidden;
    position: relative;
    padding-top: 2.5rem;
    padding-right: 1.1rem;

    &::before {
      content: "";
      position: absolute;
      background-color: #343a40;
      top: 0;
      bottom: 0;
      right: 1rem;
      left: 0;
    }
  }
  
  &__body {
    padding: 0;
    &:before {
      content: "";
      position: absolute;
      top: 0;
      left: -1.1rem;
      width: 1.1rem;
      bottom: 0;
      background-color: #fff;
      z-index: 0;
    }
  }
}


#spend-track {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

.aside {
  overflow: auto;
  margin-top: 3.5rem;
}
</style>
