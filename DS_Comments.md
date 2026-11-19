## Drop:
* AsOfDate
* Program  
because both constant
* BorrName
* BorrStreet  
because only identifiers (like unique codes ish) so have no predictive power
* BorrCity
* BorrZip  
because there are still too many distinct values since too fine-grained
* ProjectState  
see explanation below
* BankName
* BankStreet
* BankCity
* BankZip  
because other identifiers of banks, redundants since we have LocationID
* BankState  
because it is just the legal headquarter of the banks, so not really useful
* BankFDICNumber
* BankNCUANumber  
only after creating LenderType
* SBAGuaranteedApproval  
only after creating GuaranteedPct
* ApprovalDate  
only after creating the target
* FirstDisbursementDate
* TermInMonths  
for values + leakage problems
* Processing Methods
only after creating ChannelGroup
* NaicsDescription
* NaicsCode  
only after creating Score
* FranchiseName
* FranchiseCode
only after creating IsFranchise
* ProjectCounty
* SBADistrictOffice
* CongressionalDistrict
* LoanStatus
* ChargeOffDate  
only after creating the target
* PaidInFullDate
* GrossChargeOffAmount
* RevolverStatus
* CollateralInd  
only after craeting IsRevolveer and HasCollateral flags
* JobsSupported
* SoldSecMrktInd



LocationID is a code per each bank that issues the loan (so it's a number that acts as a label basically)  
Remember to import it as a string, not a number  
In the feature engineering part we could either:
* keep the labels for the top 20 banks and relable others as "Other", so grouping them or
* We could indicate a bank by its features like bank_default_rate, loans_per_bank, lender_type 
    * bank_default_rate is the default rate per each bank in the overall dataset. We must compute it from the training set only, then attach it onto the test set as well.
    * lender_type can be computed from BankFDICNumber or BankNCUANumber

BorrState we keep  
(Careful! BorrState == ProjectState 99.8% of the times, so they are basically the same thing, also intuitively, so we drop ProjectState)

BankFDICNumber e BankNCUANumber we use to create LenderType, then we drop