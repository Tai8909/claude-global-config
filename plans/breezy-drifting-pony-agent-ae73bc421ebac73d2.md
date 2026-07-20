# 回饋中心「發票自助登記」實作計畫

（完整內容見最終回覆；本檔為計畫存檔）

## 後端
- back_end/controllers/dealerController.js：新增 exports.updateMyInvoice（放在 updateMyProfile 之後）
  - 守衛：!invoiceNumber（未登記）或 invoiceStatus==='rejected' 才可寫；否則 409
  - 驗證：號碼+金額同填、金額正數、長度<=20、重複檢查排除自己
  - 原子寫入：findOneAndUpdate({_id, $or:[{invoiceNumber:{$in:[null,'']}},{invoiceStatus:'rejected'}]}, {invoiceNumber, invoiceAmount, invoiceStatus:'pending'})
- back_end/routes/dealerRoutes.js：router.put('/me/invoice', protectDealer, dealerController.updateMyInvoice)（L22 後）

## 前端
- cyberbiz_distributor_pages/distributor_dashboard.html：
  - #view-account 內「報稅資料」卡之後新增「發票登記」卡（iv- 前綴 id）
  - loadProfile() 呼叫 renderInvoiceSection(currentUser)
  - 四態 badge：未登記(!invoiceNumber)/pending/checked(有發票)/rejected
  - 可編輯僅限未登記與 rejected；即時查重重用 GET /check-invoice
  - submit → PUT /me/invoice → 更新 currentUser 並重繪

## 測試 case
未登記可登、pending 擋、checked(有發票)擋、rejected 可重填、重複發票擋、只填其一擋、金額非正數擋、登記後出現在 pending-invoices、審核通過發 10% 分潤、退回後可再登
