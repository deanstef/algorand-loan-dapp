# Algorand Beneficial Loans dApp

Algorand dApp solution for beneficial loans management between employers, employees and national authorities.

## Loan Contract Application Deployment

 1. An Employer creates a new Loan Application for its employees

        ./goal app create --creator <EMPLOYER_ADDR> --app-arg "addr:AUTHORITY_ADDR" --foreign-asset EMPLRIGHTS_ASSID --approval-prog approval_loan.teal --clear-prog clear_state_loan.teal --global-byteslices 2 --global-ints 3 --local-byteslices 0 --local-ints 5`

 2. The Authority creates an escrow account for funds deposit via a stateless Loan Escrow Contract

        ./goal clerk compile loan_escrow.teal
 
 3. The Authority realises that a new loan has been created. At this point the Authority initialises the loan 
    by funding the escrow account with a desired amount of ALGOS. Atomic Transfer:

    3.1. Create unsigned standalone transactions (AppCall + Payment txn)
    
        ./goal app call --app-id TMP_APP_ID -f <AUTHORITY_ADDR> --app-arg "str:Init" --app-arg "int:<THRESHOLD_AMOUNT>" -o loan_init.txn
        ./goal clerk send --from=<AUTHORITY_ADDR> --to=<LOAN_ESCROW_ACCOUNT> --fee=1000 --amount=<FUND_AMOUNT> --note="loan fund" --out="loan_fund.txn"
    
    3.2. Atomically group unsigned txns

        cat loan_init.txn loan_fund.txn > combinedloan_init.tx
        ./goal clerk group -i combinedloan_init.tx -o groupedloan_init.tx

    3.3. Split the unsigned txns and sign the standalone txns
    
        ./goal clerk split -i groupedloan_init.tx -o unsigned_init.tx
        ./goal clerk sign -i unsigned_init-0.tx -o init_0.stxn    
        ./goal clerk sign -i unsigned_init-1.tx -o init_1.stxn
 
    3.4. Concatenate the two signed txns
         
        cat init_0.stxn init_1.stxn > init.sgtxn
      
    3.5. Submit the signed group txn
        
        ./goal clerk rawsend -f init.sgtxn

## Loan Application Usage

1. The Employee opts-in to the application in order to initialise its local state and interact with the application calls.
To be opted in, the employee need to present the respective Employer Rights asset id. The opt-in logic verifies that the
    employee owns the rights for the specified employer.
    
        ./goal app optin --app-id APP_ID -f <EMPLOYEE_ADDR> --foreign-asset <EMPLRIGHTS_ASSID>

2. The Employer opens a loan request in behalf of an employee and sets its local variables with the details on the payslip
and the required loan. In this phase the loans is not authorised (i.e. Authorised local variable set to 0).
   
       ./goal app call --app-id APP_ID --from <EMPLOYER_ADDR> --app-account <EMPLOYEE_ADDR> --app-arg "str:Request" --app-arg "int:<PAYSLIP>" --app-arg "int:<LOAN_AMOUNT>" --app-arg "int:0"

3. The Authority checks off-chain the presence of existent loans and the KYC, for the employee. The results of this off-chain
check are then stored on the employee local state.
   
       ./goal app call --app-id APP_ID --from <AUTHORITY_ADDR> --app-account <EMPLOYEE_ADDR> --app-arg "str:Check" --app-arg "int:<AMOUNT_EXISTING_LOANS>" --app-arg "int:<KYC_FLAG>"
   
4. At this point all the variable are written on the global state and employee's local state.
Both application states can que checked with goal app read command as follows.
   
   4.1. Input for global state
       
       ./goal app read --global --app-id <APP_ID>
   output

       {
          "AuthorityAddr": {
             "tb": <ADDR_BYTECODE>,
             "tt": 1
          },
          "EmployerRightsId": {
             "tt": 2,
             "ui": 2
          },
          "LoanEscrow": {
             "tb": <ADDR_BYTECODE>,
             "tt": 1
          },
          "LoanFund": {
             "tt": 2,
             "ui": <AMOUNT>
          },
          "LoanThreshold": {
             "tt": 2,
             "ui": <THRESHOLD_VALUE>
          }
       }

   4.2. Input fot local state

       ./goal app read --local --from <EMPLOYEE_ID> --app-id <APP_ID>

   output

       {
          "AuthorityKyc": {
             "tt": 2,
             "ui": 1
          },
          "AuthorizedLoan": {
             "tt": 2
          },
          "ExistingLoan": {
             "tt": 2
          },
          "Payslip": {
             "tt": 2,
             "ui": <PAYSLIP_VAL>
          },
          "RequiredLoan": {
             "tt": 2,
             "ui": <LOAN_AMOUNT>
          }
       }

5. The Authority authorizes the loan and emits the payment through the dedicated escrow account with ALGO transfer.
Atomic Transfer:
   5.1. Create unsigned standalone transactions (AppCall + Payment txn)
       
       ./goal app call --app-id 6 -f <AUTHORITY_ADDR> --app-account <EMPLOYEE_ADDR> --app-arg "str:Authorize" -o loan_authorize.txn   
       ./goal clerk send --from=<LOAN_ESCROW_ACCOUNT> --to=<EMPLOYEE_ADDR> --fee=1000 --amount=<LOAN_AMOUNT> --note="loan payment" --out="loan_pay.txn"

   5.2. Atomically group unsigned txns

       cat loan_authorize.txn loan_pay.txn > combinedloan_auth.tx         
       ./goal clerk group -i combinedloan_auth.tx -o groupedloan_auth.tx

   5.3. Split the unsigned txns and sign the standalone txns. Transactions involving stateless ASC1 must be signed with
Logic Signature (link) providing its TEAL logic:
    
       ./goal clerk split -i groupedloan_auth.tx -o unsigned_auth.tx
       ./goal clerk sign -i unsigned_auth-0.tx -o auth_0.stxn
       ./goal clerk sign -i unsigned_auth-1.tx -p loan_escrow.teal -o auth_1.sltxn
   
    5.4. Concatenate the two signed txns
   
       cat auth_0.stxn auth_1.sltxn > auth.sgtxn
    
    5.5. Submit the signed group txn
   
       ./goal clerk rawsend -f auth.sgtxn

Once the Atomic Transfer is completed, the employee account will be credited with the required loan amount from the specific
Loan Escrow Account.