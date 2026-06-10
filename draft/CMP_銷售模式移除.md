在 CMP 2.0 中，`salesModel` 主要用於判斷銷售模式，並根據不同的銷售模式顯示不同的內容或執行不同的邏輯判斷。

#### 1. 客戶標題顯示
  在客戶資訊區塊的標題處，根據 `salesModel` 的值決定是否顯示"經銷商"標籤：

- modification/detail.component.html
  ```html
  <span *ngIf="order.header.salesModel === 'DISTRIBUTION' || order.header.salesModel === 'DISTRIBUTOR_SELF_USE'">({{ 'dealer' | translate }})</span>
  ```
- order/detail/order-customer.html
  ```html
  <span *ngIf="order.header.salesModel === 'DISTRIBUTION' || order.header.salesModel === 'DISTRIBUTOR_SELF_USE'">({{ 'dealer' | translate | titlecase }})</span>
  ```

#### 2. 客戶資料欄位是否必填
當 salesModel 為 自用(selfUse) 或 POC 時，相關欄位不強制必填；其他模式則必填。

- order-customer.component.ts
  ```typescript
  ngOnChanges(changes: SimpleChanges) {
    // ......
    if (changes['salesModel'] && (!changes['salesModel'].firstChange || isModifyMode)) {
      this.onSalesModelChange();
    }
    // ......
  }

  /** 修改 [銷售模式] 時的連動 */
  onSalesModelChange() {
    const filters: string[] = ['erpCustomerNumber', 'customer.id', 'customer.id', 'customerContact.id', 'billingContact.id', 'billingAddress'];
    const isRequired = !(this.salesModel === OrderSalesModel.selfUse || this.salesModel === OrderSalesModel.poc);
    filters.forEach(filter => {
      this.filterAttribute
        .filter(f => f.internalVariableName === filter)
        .forEach(f => {
          f.required = isRequired;
          if (!isRequired) {
            f.errorTip = '';
          }
      });
    });
  }
  ```

#### 3. 最終使用者公司名稱與地址
當 salesModel 為 經銷(distribution) 或 POC 時，子單最終使用者的 `公司名稱 (endUser.id)` 與 `地址 (endUserBillingAddress)` 為必填 。
- data-tool.component.ts
  ```typescript
  // 檢查必填欄位
  checkOrder(
    order: Order,
    oriOrder: Order,
    orderBaseStep: BaseStep,
    newData: any,
    checkOrderResault: CheckSubOrder,
    attribute: FilterAttribute[],
    isModifyMode: boolean = false,
    saveTranceNote?: Function,
  ) {
    // ......
    // 銷售模式為經銷或POC，公司名稱與地址為必填
    if (order.header.salesModel === OrderSalesModel.distribution || order.header.salesModel === OrderSalesModel.poc) {
      // 公司名稱
      if (!(newData['endUser'] && newData['endUser'].id) && attribute) {
        checkOrderResault.bodyRequiredColumn.push(this.translate.instant('enduser'));
        this.setErrorTip(attribute, 'endUser.id');
      }
      // 地址
      if (!newData['endUserBillingAddress'] && attribute) {
        checkOrderResault.bodyRequiredColumn.push(this.translate.instant('address'));
        this.setErrorTip(attribute, 'endUserBillingAddress');
      }
    }
    // ......
  }
  ```

- sub-order.component.ts

  ```typescript
  ngOnChanges(changes: SimpleChanges) {
    // ......
    // 銷售模式
    if (changes['salesModel']) {

      // 銷售模式為經銷或POC，公司名稱與地址為必填
      if (this.salesModel === OrderSalesModel.distribution || this.salesModel === OrderSalesModel.poc) {
        this.filterAttribute.filter(f => f.internalVariableName === 'endUser.id').forEach(f => f.required = true);
        this.filterAttribute.filter(f => f.internalVariableName === 'endUserBillingAddress').forEach(f => f.required = true);
      } else {
        this.filterAttribute.filter(f => f.internalVariableName === 'endUser.id').forEach(f => f.required = false);
        this.filterAttribute.filter(f => f.internalVariableName === 'endUserBillingAddress').forEach(f => f.required = false);
      }

      // 銷售模式為「直銷」或「經銷自用」，子單的「公司名稱」、「帳單聯絡人」、「其他聯絡人」、「地址」欄位改為灰階不可填寫
      const noCompanySituation = this.salesModel === OrderSalesModel.directSelling || this.salesModel === OrderSalesModel.distributionSelfUse;
      const companyInfos = ['endUser.id', 'endUserContact.id', 'endUserOtherContact', 'endUserBillingAddress'];
      companyInfos.forEach(info => {
        this.filterAttribute.filter(f => f.internalVariableName === info).forEach(f => f.disabled = noCompanySituation);
      });
    }
  }

  setCustomRequiredAttribute() {
    // 銷售模式為經銷或POC，公司名稱與地址為必填
    if (this.orderHeader.salesModel === OrderSalesModel.distribution || this.orderHeader.salesModel === OrderSalesModel.poc) {
      // 公司名稱
      if (!(this.subOrder['endUser'] && this.subOrder['endUser'].id)) {
        this.filterAttribute.filter(f => f.internalVariableName === 'endUser.id').forEach(f => {
          f.hasFeedback = true;
          f.errorTip = this.translate.instant('This field is required!');
        });
      } else {
        this.filterAttribute.filter(f => f.internalVariableName === 'endUser.id').forEach(f => {
          f.errorTip = '';
        });
      }

      // 地址
      if (!this.subOrder['endUserBillingAddress']) {
        this.filterAttribute.filter(f => f.internalVariableName === 'endUserBillingAddress').forEach(f => {
          f.hasFeedback = true;
          f.errorTip = this.translate.instant('This field is required!');
        });
      } else {
        this.filterAttribute.filter(f => f.internalVariableName === 'endUserBillingAddress').forEach(f => {
          f.errorTip = '';
        });
      }
    }
  }
  ```


#### 4. 微軟相關檢查
在訂單修改檢查功能中，需根據 salesModel 及訂單狀態執行微軟相關檢查：

檢查產品 U 數
- order/sub-order/products/products.component.ts
  ```typescript
  /** 微軟檢查產品 U 數 */
  checkProducts() {
    // ......
    const data: {
      cloudId: string,
      products: OrderProduct[],
      originalInfo: any,
      salesModel: string
    } = {
      //......
      salesModel: this.salesModel,
    };

    (this.dataProvider as MicrosoftDataProviderService).checkProducts(data).subscribe({
      // ......
    });
  }

- modification/detail.component.ts
  ```typescript
  /** 微軟檢查產品 U 數 */
  checkProducts(subOrder: SubOrder, index: number) {
    // ......
    const data: {
      cloudId: string,
      products: OrderProduct[],
      originalInfo: any,
      salesModel: string
    } = {
      // ......
      salesModel: this.order.header.salesModel,
    };

    this.microsoftDataSvc.checkProducts(data).subscribe(
      // ......
    )
  }
  ```
- microsoft-data-provider.service.ts  
  ```typescript
  checkProducts(data: { cloudId: string, products: OrderProduct[], originalInfo: any, salesModel: string }): Observable<boolean> {
    return new Observable(sub => {
      this.ui.isChecking = true;
      this.ui.isChecked = undefined;

      this.orderSvc.checkProducts(data).subscribe({
        // ...呼叫後端API檢查產品，並傳入salesModel參數...
      })
    })
  }
  ```
- order.service.ts
  ```typescript
    /** 檢查產品 U 數 */
    checkProducts(data: { cloudId: string, products: OrderProduct[], originalInfo: any, salesModel: string }): Observable<ResponseData> {
      return this.api.post(this.gateway.order + 'Microsoft/checkProducts', new RequestData(data));
    }
  ```

partnerId 欄位是否必填
- data-tool.component.ts
  ```typescript
  // partner id，PM之後的階段都是必填
  const getPartnerIdVal = this.scanDataSvc.getValue(newData, ('originalInfo.partnerId').split('.'));
  const isRequiredStatus =
    orderBaseStep.currentStatus === ApprovalStatus.reviewPm ||
    orderBaseStep.currentStatus === ApprovalStatus.reviewPurchase;
  if (order.header.salesModel === OrderSalesModel.distribution && !getPartnerIdVal && attribute && isRequiredStatus) {
    checkOrderResault.bodyRequiredColumn.push('partnerId');
    this.setErrorTip(attribute, 'originalInfo.partnerId');
  }
  ```
- microsoft-extra-field.component.ts
  ```typescript
  setPartnerIdAttributes() {
    const isRequiredStatus =
      this.orderStep.buttonList.some(btn => btn === OrderButtonStep.next) &&
      this.orderStep.currentStatus !== ApprovalStatus.draft &&
      this.orderStep.currentStatus !== ApprovalStatus.rejected &&
      this.orderStep.currentStatus !== ApprovalStatus.drawn &&
      this.orderStep.currentStatus !== ApprovalStatus.update;

    this.filterAttribute
      .filter(f => f.internalVariableName === 'originalInfo.partnerId')
      .forEach(f => {
        f.required = this.salesModel === OrderSalesModel.distribution && isRequiredStatus;
      });
  }
  ```

取得partnerId
- microsoft-extra-field.component.ts
  ```typescript
  getPartnerId() {
    if (!(this.customerId && (this.salesModel === OrderSalesModel.distribution || this.salesModel === OrderSalesModel.distributionSelfUse || this.salesModel === OrderSalesModel.poc))) {
      return;
    }
    let filter = new Filter();
    filter.and.push({
      field: 'customerId',
      comparator: Comparator.equal,
      value: this.customerId,
    });
    // ......
  }
  ```

#### 5. 複製子公司訂單
  複製子公司訂單時，限制經銷/經銷自用模式不可複製

- order/detail.component.ts
  ```typescript
  showCloneOrderModal() {
    // ......
    if (this.order.header.salesModel === OrderSalesModel.distribution || this.order.header.salesModel === OrderSalesModel.distributionSelfUse) {
      this.notify.warning('', this.translate.instant('salesModel cannot be distribution'));
      modal.destroy();
      this.ui.isLoading = false;
      return;
    }
    // ......
  }
  ```

#### 6. 報價單匯入
取得報價單時，根據 salesModel 決定對應的 CRM 客戶欄位

- order/detail.component.ts
  ```typescript
  getQuotation(): Promise<boolean> {
    // ......
    const crmCustomer =
      this.order.header.salesModel === OrderSalesModel.directSelling && res.data.sellMode === OrderSalesModel.directSelling
        ? res.data.enduserInfoId
        : res.data.distributorId;
    // ......
  }
  ```
