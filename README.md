@RestResource(urlMapping='/customer-support/ocm/case-list')
global With Sharing class WS_OCMCaseList {
    
    Public Static final String ISO_DATE_FORMAT ='yyyy-MM-dd\'T\'HH:mm:ss.SSS\'Z\'';  
    public static String userId;
    Private Static FINAL String ECLIPSE = Label.CLOCT12CCC07;
    Private Static FINAL String CASSINI = Label.CLJUN16CSN173;
    Private Static FINAL String NEWSTATUS = 'New';
    Private Static FINAL String INPROGSTAUS = 'In Progress';
    Private Static FINAL String CLOSEDSTATUS = 'Closed';
    Private Static FINAL String CANCELLEDSTATUS = 'Cancelled';
    Private Static FINAL String ANSPROVIDEDTOCUSTOMER = 'Answer Provided to Customer';
    Private Static FINAL String CUSTOMER = 'Customer';
    Private Static FINAL String SHOPSE = 'Shop SE';
    Private static final string COMMUNITYSOURCE = 'community-source';
    Public static string reqIDMSToken = '';
    global static WS_OKTAAPIHandler.OktaGetUserWrapper currentUser = new WS_OKTAAPIHandler.OktaGetUserWrapper();
    
    //Get Method
    @HttpGet 
    global static void getCaseList(){
        
        String format;
        String language;
        Integer pageSizeInt;
        Integer pageNumberInt;
        String serviceContractId;
        String accountId;
        String contractId;
        String keywordSearch;
        DateTime dateTimeStart;
        DateTime dateTimeEnd;
        String sortedBy;
        String sortedOrder;
        String contactExtendedApps = '';
        String SORTEDBYSTR = 'sorted-by';
        String SORTEDORDERSTR = 'sorted-order';
        String SERVICECONTRACTIDSTR = 'service-contractid';
        String CREATEDSTART = 'createddate-start';
        String CREATEDEND = 'createddate-end';
        String KEYWORDSEARCHSTR = 'keyword-search';
        Integer totalNumberOfPages;
        Integer totalNumberOfCases;
        Decimal totalNumberPage;
        Set<String> setCommunitySource = new Set<String>();
        Set<String> statusFilter = new Set<String>();
        Map<String,String> mapReturnData = new Map<String,String>();
        List<CaseDetailWrapper> lstCasesWrapper = new List<CaseDetailWrapper>();
        CaseViewResults caseViewResultsWrapper = new CaseViewResults();
        CaseListAPIResponse apiResponse = new CaseListAPIResponse();
        RestResponse restResponse = RestContext.response;
        restResponse.addHeader('Content-Type', 'application/json;charset=UTF-8');
        RestRequest request = RestContext.request;
        Map<String, String> headers = request.headers;
        
        
        Map<String,String> mapStateCodeStateName = new map<String,String>();
        
		   try{
            reqIDMSToken = headers.containsKey('X-IDMS-Authorization') ? headers.get('X-IDMS-Authorization') : '';
            System.debug('reqIDMSToken--->'+reqIDMSToken);
            if(reqIDMSToken != ''){
                if((reqIDMSToken).startsWith('00D')){
                    if(!test.isRunningTest()){
                        userId = WS_OCMAPIHandler.getUserInfoForToken(reqIDMSToken);
                    }
                    if(String.isNotBlank(userId)){
                        currentUser = WS_OKTAAPIHandler.getDetailsOfLoggedInUserWrapper(userId);
                    }
                }
                else {
                    if(!test.isRunningTest()){
                        String idpToken = reqIDMSToken;
                        currentUser = WS_OKTAAPIHandler.OktaGetUsersAPI(idpToken);
                    }
                }
            }
            
            
                
                if(request.params.containsKey('format')){
                    if (String.isNotBlank(request.params.get('format'))){
                        format = request.params.get('format');
                    }      
                    else{
                        format = 'List';
                    }
                }
                else{ 
                    throw new AP_CCCaseServiceExpectedException (400,'13','Required mandatory parameter is missing: format');
                }
               
                if(request.params.containsKey('language')){
                    if (String.isNotBlank(request.params.get('language'))){
                        String api_Language =  request.params.get('language').substring(0,2);
                        api_Language = api_Language+ '_' + currentUser.Countrycode;
                        language = WS_OCMCaseCategories.processLanguages(api_Language);
                        language = validateLanguageCode(request.params.get('language'));
                    }      
                    else{
                        language = 'en_US';
                    }
                }
                else{ 
                    language = 'en_US';
                }
                if(request.params.containsKey(COMMUNITYSOURCE) && String.isNotBlank(request.params.get(COMMUNITYSOURCE))){
                      setCommunitySource = new Set<String>(String.valueOf(request.params.get(COMMUNITYSOURCE)).split(','));
                        
                      if(!validateCommunitySource(setCommunitySource)){
                          throw new AP_CCCaseServiceExpectedException (400,'13','Invalid Community Source: community-source');
                      }
                }
                
                if((request.params.containsKey('page-size')) || (format == 'Export')){
                    if (String.isNotBlank(request.params.get('page-size'))){
                        pageSizeInt = Integer.valueOf(request.params.get('page-size'));
                        if(pageSizeInt > 1000){
                            pageSizeInt = 1000;
                        }
                    }    
                    else{
                        pageSizeInt = 10;
                    }    
                }
                else{
                    throw new AP_CCCaseServiceExpectedException (400,'13','Required mandatory parameter is missing: page-size');
                }
                if(request.params.containsKey('page-number') || (format == 'Export')){
                    if (String.isNotBlank(request.params.get('page-number'))){
                        pageNumberInt = Integer.valueOf(request.params.get('page-number'));
                    }
                    else{
                        pageNumberInt = 1;
                    }
                }
                else{
                    throw new AP_CCCaseServiceExpectedException (400,'13','Required mandatory parameter is missing: page-number');
                }
                if(request.params.containsKey(SORTEDBYSTR) && String.isNotBlank(request.params.get(SORTEDBYSTR))){
                    sortedBy = request.params.get(SORTEDBYSTR);
                }
                else{
                     throw new AP_CCCaseServiceExpectedException (400,'13','Required mandatory parameter is missing: sorted-by');
                }
                if(request.params.containsKey(SORTEDORDERSTR)&&String.isNotBlank(request.params.get(SORTEDORDERSTR))){
                    sortedOrder = request.params.get(SORTEDORDERSTR);
                }
               	else{
                    throw new AP_CCCaseServiceExpectedException (400,'13','Required mandatory parameter is missing: sorted-order');
                }
				if(request.params.containsKey('status-filter')){
                    if(String.isNotBlank(request.params.get('status-filter'))){
                        statusFilter = new Set<String>(String.valueOf(request.params.get('status-filter')).split(','));
                    }                    
                    else{
                        statusFilter.add('02');
                    }
                }
                else{
                    throw new AP_CCCaseServiceExpectedException (400,'13','Required mandatory parameter is missing: Status Filter');
                }
                if(request.params.containsKey(SERVICECONTRACTIDSTR) && String.isNotBlank(request.params.get(SERVICECONTRACTIDSTR))){
                    serviceContractId = request.params.get(SERVICECONTRACTIDSTR);
                }
                if(request.params.containsKey('accountid') && String.isNotBlank(request.params.get('accountid'))){
                    accountId = request.params.get('accountid');
                }
                System.debug('!!!! accountId : ' + accountId);
                
                
                //Read the Input Parameter contractid
                if(request.params.containsKey('contractid') && String.isNotBlank(request.params.get('contractid'))){
                    contractId = request.params.get('contractid');
                }
                System.debug('!!!! contractId : ' + contractId);
                
                
                //Read the Input Parameter createddate-start
                if(request.params.containsKey(CREATEDSTART) && String.isNotBlank(request.params.get(CREATEDSTART))){                    
                    dateTimeStart = DateTime.valueOfGMT(request.params.get(CREATEDSTART) + ' 00:00:00.000Z');
                }
                System.debug('!!!! dateTimeStart : ' + dateTimeStart);
                
                
                //Read the Input Parameter createddate-end
                if(request.params.containsKey(CREATEDEND) && String.isNotBlank(request.params.get(CREATEDEND))){
                        dateTimeEnd = DateTime.valueOfGMT(request.params.get(CREATEDEND) + ' 23:59:59.000Z');
                }
                System.debug('!!!! dateTimeEnd : ' + dateTimeEnd);
                
                
                //Read the Input Parameter keyword-search
                if(request.params.containsKey(KEYWORDSEARCHSTR) && String.isNotBlank(request.params.get(KEYWORDSEARCHSTR))){
                        keywordSearch = request.params.get(KEYWORDSEARCHSTR);
                }
                System.debug('!!!! keywordSearch : ' + keywordSearch);
                
                // Get details of current user
                
                contactExtendedApps = currentUser.ContactExtendedApplications;
                System.debug('!!!! contactExtendedApps : ' + contactExtendedApps);
                
                // validate account and contact details
            	if(currentUser.contactId!=null){
                	mapReturnData = WS_OKTAAPIHandler.validateContactAccountDetails(currentUser,headers);
                	System.debug('!!!! mapReturnData : ' + mapReturnData);
                }else{
                    mapReturnData.put('isEligible','true');
                }
                
                if(mapReturnData.get('isEligible') == 'true') {
                    
                    for(StateProvince__c objState : [SELECT Name, StateProvinceCode__c, CountryCode__c FROM StateProvince__c]) {
                        mapStateCodeStateName.put(objState.StateProvinceCode__c + '-' + objState.CountryCode__c, objState.Name);
                    }
                    
                    // list of Cases, Accounts and Contracts related to current user
                    caseViewResultsWrapper = getAllAPICases(currentUser, contactExtendedApps, setCommunitySource, sortedBy, sortedOrder, pageSizeInt, 
                                                            pageNumberInt, contractId, accountId, dateTimeStart, dateTimeEnd, 
                                                            keywordSearch, statusFilter,format,serviceContractId);
                    System.debug('!!!! caseViewResultsWrapper : ' + caseViewResultsWrapper);
                    
                    if(caseViewResultsWrapper != null && caseViewResultsWrapper.totalNumberOfPages != null && caseViewResultsWrapper.totalNumberOfCases != null) {
                        
                        totalNumberOfPages = caseViewResultsWrapper.totalNumberOfPages;
                        totalNumberOfCases = caseViewResultsWrapper.totalNumberOfCases;
                        totalNumberPage = totalNumberOfCases/pageSizeInt;
                        
                        System.debug('!!!! totalNumberOfPages : ' + totalNumberOfPages);
                        System.debug('!!!! totalNumberOfCases : ' + totalNumberOfCases);
                        System.debug('!!!! totalNumberPage : ' + totalNumberPage);
                        
                        lstCasesWrapper = populateCaseFieldsMethod(caseViewResultsWrapper.lstAPICases,format,mapStateCodeStateName,pageNumberInt,totalNumberOfPages,currentUser,language);
                        
                        System.debug('!!!! lstCasesWrapper : ' + lstCasesWrapper);
                    }
                    
                    // Returning the Response
                    if(lstCasesWrapper != null){
                        apiResponse.cases = lstCasesWrapper;
                    }
                    apiResponse.totalNumberOfPages = totalNumberOfPages;
                    apiResponse.totalNumberOfCases = totalNumberOfCases;
                    
                    restResponse.statusCode = 200;
                    restResponse.responseBody = Blob.valueOf(JSON.serializePretty(apiResponse, true));                
                }
            //}
        }
        catch (AP_CCCaseServiceExpectedException ex) {
            restResponse.statusCode = ex.statusCode;               
            WS_OCMAPIHandler.OCMErrorWrapper[] errorDetailWrapper = new WS_OCMAPIHandler.OCMErrorWrapper[]{ 
                new WS_OCMAPIHandler.OCMErrorWrapper(Integer.valueof(ex.errorCode), ex.message)
                    };                        
                        restResponse.responseBody = Blob.valueOf(JSON.serializePretty(errorDetailWrapper, true));
            System.debug('!!!! restResponse.responseBody '+ restResponse.responseBody);  
        } 
        catch (AP_CCCCustomExpectedException ex) {
            system.debug('Execption>> '+ex);
            System.debug('!!!! Catch message : '+ex.getMessage());  
            if(!test.isRunningTest()){
                restResponse.statusCode = ex.statusCode; 
            }
            WS_OCMAPIHandler.OCMErrorWrapper[] errorDetailWrapper = new WS_OCMAPIHandler.OCMErrorWrapper[]{ 
                new WS_OCMAPIHandler.OCMErrorWrapper(Integer.valueof(ex.code), ex.message)
                    }; 
            restResponse.responseBody = Blob.valueOf(JSON.serializePretty(errorDetailWrapper, true));
        } 
        catch (Exception e) {
            restResponse.statusCode = 500;
            //CCCOCM-1613
            string operation = ' - Case List';
            string debugLog = WS_OCMAPIHandler.createDebugLog(e.getStackTraceString(), e.getMessage(), restResponse.statusCode , 'Partner Case', request.httpMethod, operation); 
            String  techErrorMessage='Technical issue has occured'; 
            WS_OCMAPIHandler.OCMErrorWrapper[] errorDetailWrapper = new WS_OCMAPIHandler.OCMErrorWrapper[]{ 
                new WS_OCMAPIHandler.OCMErrorWrapper(10,'Technical error, '+techErrorMessage+','+debugLog)
                    };
                        restResponse.responseBody = Blob.valueOf(JSON.serializePretty(errorDetailWrapper, true));
            System.debug('!!!! restResponse.responseBody '+ restResponse.responseBody);
        }
    }
    
    public static List<CaseDetailWrapper> populateCaseFieldsMethod(List<Case> lstAPICases,String format,Map<String,String> mapStateCodeStateName,
                                                                   Integer pageNumberInt,Integer totalNumberOfPages, WS_OKTAAPIHandler.OktaGetUserWrapper currentUser,String language)
    {
        
        List<CaseDetailWrapper> lstCasesWrapper = new List<CaseDetailWrapper>();
        String accountConcat = '';
        String mapCaseKey;
        String hotCase = System.Label.CLR417PAHotCaseTeamList;
        Set<String> caseCategoryValuesSet = new Set<String>(); 
        Set<String> caseReasonValuesSet = new Set<String>();
        Set<Id> caseIdSet = new Set<Id>();
        Set<Id> caseOwnerIdSet = new Set<Id>();
        if (pageNumberInt <= totalNumberOfPages && !lstAPICases.isEmpty()) {
        	for(Case objCase : lstAPICases) {
                caseIdSet.add(objCase.Id);
                caseOwnerIdSet.add(objCase.OwnerId);
                caseCategoryValuesSet.add(objCase.SupportCategory__c);
                caseReasonValuesSet.add(objCase.Symptom__c);
        	}
        }
        String idmsAil ;
        if(reqIDMSToken.startsWith('00D')){
            idmsAIL = currentUser.IDMSAil;
        }else{
            if(!test.isRunningTest()){
            	idmsAIL = AP_OCMIdentityAILCallout.getAILDetails(reqIDMSToken,currentUser.federatedId); 
            }
        }
        
        Map<Id,String> mapCaseCategory	= getCustomFacingCategory(caseCategoryValuesSet,caseReasonValuesSet,language,lstAPICases);
        Map<Id,AP_CaseCommentHandler.CaseOwnerDisplay> caseAgentNameWrapper = AP_CaseCommentHandler.getAgentInfo(caseIdSet,caseOwnerIdSet);
        if (pageNumberInt <= totalNumberOfPages && !lstAPICases.isEmpty()) {
            for(Case objCase : lstAPICases) {
                
                CaseDetailWrapper caseRecord = new CaseDetailWrapper();
                
                caseRecord.caseId = objCase.Id;
                caseRecord.caseNumber = objCase.CaseNumber;
                caseRecord.communitySource = (objCase.CommunitySource__c == null) ? '' : objCase.CommunitySource__c ;
                caseRecord.subject = objCase.Subject;
                caseRecord.contactId = objCase.ContactId;
                if(objCase.ContactId == currentUser.ContactId || objCase.CommunitySource__c == 'PA'){
                    caseRecord.contactName = objCase.Contact.Name;
                }
                else{
                    caseRecord.contactName = '';
                }
                caseRecord.accountId = objCase.AccountId;
                caseRecord.accountName = objCase.Account.Name;
                caseRecord.contractName = (objCase.ContractName__c == null) ? '' : objCase.ContractName__c ;
                caseRecord.contractId = (objCase.RelatedContract__c == null) ? '' : objCase.RelatedContract__c ;
                caseRecord.consumptionOfPoints = String.valueOf(objCase.Points__c) ;
                caseRecord.product = (objCase.TECH_ProductName__c != null) ? objCase.TECH_ProductName__c : ''; 
                caseRecord.customerStatus = objCase.PAGCS_Web_Status__c;
                caseRecord.origin = objCase.Origin;
                
                caseRecord.caseResolvingAgentName = caseAgentNameWrapper.containsKey(objCase.Id) ? String.isNotBlank(caseAgentNameWrapper.get(objCase.Id).caseOwnerName) ? caseAgentNameWrapper.get(objCase.Id).caseOwnerName : '' : '';
                caseRecord.caseResolvingAgentId = caseAgentNameWrapper.containsKey(objCase.Id) ? String.isNotBlank(caseAgentNameWrapper.get(objCase.Id).caseOwnerId) ? caseAgentNameWrapper.get(objCase.Id).caseOwnerId : '' : '';
                
                caseRecord.createdDate = (objCase.createdDate != null) ? objCase.createdDate.formatGMT(ISO_DATE_FORMAT) : '' ;               
                caseRecord.lastModifiedOn = (objCase.LastModifiedDate != null) ? objCase.LastModifiedDate.formatGMT(ISO_DATE_FORMAT) : '' ;
                caseRecord.closedDate =  (objCase.ClosedDate != null) ? objCase.ClosedDate.formatGMT(ISO_DATE_FORMAT) : '' ;
                caseRecord.answerProvidedToCustomer = objCase.AnswerToCustomer__c;
                caseRecord.description = objCase.CustomerRequest__c;
                
                caseRecord.CustomerFacingcategory = mapCaseCategory.containsKey(objCase.Id) ? mapCaseCategory.get(ObjCase.Id) : '';
                // PA related data
                if(String.isNotBlank(objCase.Account.BillingState)) { 
                    
                    accountConcat = objCase.Account.BillingCountry + ' - ' + objCase.Account.Name + 
                        ' ' + '(' + objCase.Account.BillingCity + ')' +  ', ' + ' ' + 
                        mapStateCodeStateName.get(objCase.Account.BillingState + '-' + objCase.Account.BillingCountry);
                }
                else {
                    accountConcat = objCase.Account.BillingCountry + ' - ' + objCase.Account.Name + 
                        ' ' + '(' + objCase.Account.BillingCity + ')';
                }
                caseRecord.paAccountName = accountConcat;
                caseRecord.accountSEAccountId = objCase.Account.SEAccountId__c;
                caseRecord.paContactCfid = objCase.CTRValueChainPlayer__r.LegacyPIN__c;
                caseRecord.paCaseType = objCase.PACaseType__c;
                
                if(objCase.CustomerMajorIssue__c == 'Yes' || objCase.CustomerMajorIssue__c == 'To be reviewed' ||
                   (String.isNotBlank(objCase.Team__c) && hotCase.contains(objCase.Team__c))){
                       caseRecord.paHotCase = 'Yes';
                   }    
                else{
                    caseRecord.paHotCase = 'No';
                }
                caseRecord.paCustomerInternalCaseNumber = objCase.PACustomer_internal_case_number__c;
                caseRecord.paTargetDate = String.valueOfGmt(objCase.CustomerRequestedDate__c);
                caseRecord.alarmDateAndTime = (objCase.Alarm_date_and_time__c != null) ? objCase.Alarm_date_and_time__c.formatGMT(ISO_DATE_FORMAT) : '' ;
                caseRecord.siteName = objCase.Site__r.Name;
                caseRecord.siteGuid = objCase.Site__r.LegacySiteId__c;
                caseRecord.priority = objCase.Priority;                
                //CCCOCM-1603               
                caserecord.CustomerRequest = objCase.CustomerRequest__c;
                caserecord.salesOrderNumber = objCase.SalesOrderNumber__c;    
       			caserecord.InstalledAtAccountName = objCase.AssetOwner__r.Name;	   
                caserecord.InstalledProductName = objCase.SVMXC__Component__r.Name;
                caserecord.caseTracker = (currentUser.ContactId == objCase.ContactId) ? 'My Cases' : 'Other\'s Cases';
                //Check for PS User   
                
                
                if(idmsAIL.contains('(Feature;PremiumSupport)') && objCase.SVMXC__Service_Contract__r.HideFromCustomerView__c == false){
                //Check for PS User              
                if(idmsAIL.contains('(Feature;PremiumSupport)') && objCase.SVMXC__Service_Contract__r.HideFromCustomerView__c == false){
                	caserecord.ServiceLine = objCase.SVMXC__Service_Contract__r.Name;
                	caserecord.ServiceLine = objCase.SVMXC__Service_Contract__r.Name;
                }                
                caserecord.ServiceLine = objCase.SVMXC__Service_Contract__r.Name;        
                }  
                
                if(String.valueOfGmt(objCase.FLRTargetDate__c) != null) {
                    caseRecord.firstLogicalResponseTargetDate = objCase.FLRTargetDate__c.formatGMT(ISO_DATE_FORMAT);
                } else {
                    caseRecord.firstLogicalResponseTargetDate = '';  
                }
                if (objCase.ServiceLine__c != null && idmsAIL.contains('(Feature;PremiumSupport)')){
                    caseRecord.serviceContractId = objCase.ServiceLine__c;
                } else {
                    caseRecord.serviceContractId = '';
                }
                if (objCase.ServiceLine__r.Name != null && idmsAIL.contains('(Feature;PremiumSupport)')){
                    caseRecord.serviceContractName = objCase.ServiceLine__r.Name;
                } else {
                    caseRecord.serviceContractName = ''; 
                }
                lstCasesWrapper.add(caseRecord);
            }
        }
        return lstCasesWrapper;
    }
	public static CaseViewResults getAllAPICases(WS_OKTAAPIHandler.OktaGetUserWrapper loggedInUser, String contactExtendedApps, Set<String> setCommunitySource, String sortedBy, 
                                                 String order, Integer pageSize, Integer pageNumber, String contractId, 
                                                 String accountId, DateTime dateTimeStart, DateTime dateTimeEnd, 
                                                 String keywordSearch, set<String> statusFilter, String format, String serviceContractId) 
    {
        system.debug('setCommunitySource--->'+setCommunitySource);
        CaseViewResults caseViewResWrapper = new CaseViewResults();
        OCMSortByValuesMapping__c mapOCMCustomSetting = OCMSortByValuesMapping__c.getValues(sortedBy);
        List<Case> lstApiCases = new List<Case>();
        List<Case> lstAllCases = new List<Case>();
        List<String> beneficiaryEclipseContractIds = new List<String>();
        List<String> adminEclipseContractIds = new List<String>();  
        List<String> eclipseContractIds = new List<String>();  
        List<String> cassiniContractIds = new List<String>();
        List<String> cvcpServiceContractIds = new List<String>();
        Id userContactId = loggedInUser.ContactId;
        Date cutOffDate;
        Id userAccountId = loggedInUser.AccountId;
        String searchString = '%' + keywordSearch + '%';
        String caseTracker = 'Case Tracker';
        String countCasesString;
        Decimal totalNumberPage;
        Double  totalCaseCount;
        Integer totalNumberOfPages = 0;
        Integer totalNumberOfCases = 0;
        String strPA = 'PA';
        String premiumSupport = 'Premium Support';
        Set<String> setPAAccountIds = new Set<String>();
        List<String> extendedAppsList = new List<String>();
        Boolean enteredConditions = false;
        String caseListQueryString = '';
        String whereConditionQueryString = '';
        Map<String,Boolean> mapIsPremiumSupport = New  Map<String,Boolean>();
        Datetime cutOffDateTime;
        String fedId = loggedInUser.federatedId;
        Boolean userUnEnrolled = WS_OKTAAPIHandler.checkUserEnrollment(loggedInUser);
        
        //OKTA Accountid is blank, get the accountId from bFO
        if((String.isBlank(userAccountId) || userAccountId == null) && userContactId!=null){
            List<Contact> userContactLst = WS_OKTAAPIHandler.getDetailsOfLoggedInContact(userContactId);
            userAccountId = userContactLst[0].AccountId;
        }
        if(String.isNotBlank(contactExtendedApps))
            extendedAppsList = contactExtendedApps.split(';');
        
        if(loggedInUser.ContactAccountPRMCTCasesCutoffDate != null) {
            cutOffDate = loggedInUser.ContactAccountPRMCTCasesCutoffDate;          
            cutOffDateTime = Datetime.newInstance(cutOffDate.year(), cutOffDate.month(), cutOffDate.day());
        }
        system.debug('cutOffDateTime--->'+cutOffDateTime);
        caseListQueryString  = 'SELECT Id, CaseNumber, CommunitySource__c, Subject, ContactId, Contact.Name, AccountId, ClosedDate, CreatedById,OwnerId,' +
            'Account.Name, ContractName__c, RelatedContract__c, Points__c, LastModifiedDate,FLRTargetDate__c, ' +
            'PAGCS_Web_Status__c,TECH_ProductName__c,toLabel(Origin),Account.SEAccountId__c, PACaseType__c, Site__r.LegacySiteId__c, CustomerMajorIssue__c, PACustomer_internal_case_number__c, '+
            'CustomerRequestedDate__c, Team__c, Alarm_date_and_time__c, Site__r.Name, Priority, ServiceLine__r.Name '+
            ',CreatedDate, CTRValueChainPlayer__r.LegacyPIN__c, Account.BillingState, Account.BillingCountry, Account.BillingCity, AnswerToCustomer__c'+
            ',ServiceLine__c,SVMXC__Service_Contract__r.Name,SVMXC__Service_Contract__r.HideFromCustomerView__c,Symptom__c'+
            ',CustomerRequest__c,CommercialReference_lk__c,SalesOrderNumber__c,SupportCategory__c,SVMXC__Component__r.Name,AssetOwner__r.Name';
        
        countCasesString = 'SELECT Id,CaseNumber FROM Case WHERE Spam__c = False ';
        
        caseListQueryString +=  ' FROM Case WHERE Spam__c = False ' ;
        
        
        System.debug('!!!! userContactId : ' + userContactId);
        System.debug('!!!! useraccountId : ' + useraccountId);
        
        if(String.isNotBlank(accountId)){
            whereConditionQueryString += 'AND AccountId = :accountId ';
        }
        if (String.isNotBlank(contractId)){
            whereConditionQueryString += ' AND RelatedContract__c = :contractId ';
        }
        if(String.isNotBlank(serviceContractId)){
            whereConditionQueryString += 'AND ServiceLine__c = :serviceContractId ';
        }                                                  
        if (dateTimeStart != null ){
            whereConditionQueryString += ' AND CreatedDate >= :dateTimeStart ';
        }
        if (dateTimeEnd != null ){
            whereConditionQueryString += ' AND CreatedDate <= :dateTimeEnd ';
        }
        if(String.isNotBlank(keywordSearch)){
            whereConditionQueryString += ' AND ((Subject LIKE :searchString) OR (Account.Name LIKE :searchString) ' + 
                'OR(Contact.Name LIKE :searchString) OR (ContractName__c LIKE :searchString) OR (PAGCS_Web_Status__c LIKE :searchString) ' + 
                'OR (CommercialReference_lk__r.Name LIKE :searchString) OR (Family_lk__r.Name LIKE :searchString) '+
                'OR (CaseNumber LIKE :searchString) OR (ServiceLine__r.Name LIKE :searchString)) ' ;
        }
        System.debug('!!!! statusFilter : ' + statusFilter);
        whereConditionQueryString += getStatusFilterQueryString(statusFilter,userContactId,loggedInUser.ContactCaseTrackerAccess);
        System.debug('!!!! whereConditionQueryString : ' + whereConditionQueryString);
        // Checking of contactExtendedApps access 
        
        whereConditionQueryString += ' AND ( ';                                         
        
        if((setCommunitySource.contains('OCM') || setCommunitySource.isEmpty()) ) {
               
               whereConditionQueryString += '(( CommunitySource__c = ' + '\'' + '\'' + ' OR CommunitySource__c =: caseTracker ) ';
               
               if(cutOffDateTime != null) {
                   whereConditionQueryString += 'AND CreatedDate >=: cutOffDateTime ';
               }
            	
                if(userUnEnrolled){
                    whereConditionQueryString += 'AND Tech_FedId__c =: fedId) OR ';
                }
            
            	if(!userUnEnrolled && String.isBlank(loggedInUser.ContactCaseTrackerAccess)){
                    whereConditionQueryString += 'AND Tech_FedId__c =: fedId) OR';
                    enteredConditions = true;
            	} 
               
               // My Cases
               if(loggedInUser.ContactCaseTrackerAccess == Label.CLR617CCCMyCases) {
                   whereConditionQueryString += 'AND  ((ContactId = :userContactId AND AccountId = :userAccountId) OR Tech_FedId__c =: fedId)) OR ';
                   enteredConditions = true;
               }
                
               // All Cases
               if(loggedInUser.ContactCaseTrackerAccess == Label.CLR617CCCAllCases) {
                   whereConditionQueryString += 'AND AccountId = :userAccountId ) OR';
                   enteredConditions = true;
               }
            
           }                    
        
        if(setCommunitySource.contains(Label.CLOCT12CCC07) || setCommunitySource.contains('OCM') || setCommunitySource.isEmpty()) { // Eclipse
            
            if(extendedAppsList.contains(Label.CLOCT12CCC07)) {
                
                beneficiaryEclipseContractIds = WS_OKTAAPIHandler.getContractIdForBeneficiaryCVCPs(loggedInUser);
                adminEclipseContractIds = WS_OKTAAPIHandler.getContractIdForAdminCVCPs(loggedInUser);
                eclipseContractIds.addAll(beneficiaryEclipseContractIds);
                eclipseContractIds.addAll(adminEclipseContractIds);
                System.debug('!!!! eclipseContractIds : ' + eclipseContractIds);
                
                whereConditionQueryString += ' ( CommunitySource__c = :ECLIPSE AND RelatedContract__c IN :eclipseContractIds ) OR';
                
                enteredConditions = true;
            }
            else {
                whereConditionQueryString += ' ( CommunitySource__c = :ECLIPSE AND ContactId = :userContactId ) OR'; 
                enteredConditions = true;
            }
        }
        
        if(setCommunitySource.contains(Label.CLJUN16CSN173) || setCommunitySource.contains('OCM') || setCommunitySource.isEmpty()) { // Cassini
            
            if(extendedAppsList.contains(Label.CLJUN16CSN173)) { 
                
                cassiniContractIds = WS_OKTAAPIHandler.getContractIdForCassiniCVCPs(loggedInUser);
                System.debug('!!!! cassiniContractIds : ' + cassiniContractIds);
                
                whereConditionQueryString += ' ( CommunitySource__c = :CASSINI AND RelatedContract__c IN :cassiniContractIds ) OR';
                enteredConditions = true;
            }
            else {
                whereConditionQueryString += ' ( CommunitySource__c = :CASSINI AND ContactId = :userContactId ) OR';
                enteredConditions = true;
            }
        }
        
        if((setCommunitySource.contains(strPA) || setCommunitySource.isEmpty()) && extendedAppsList.contains(strPA) ) {
            
            for(AccountContactRelation objAccConRelation : [SELECT AccountId FROM AccountContactRelation 
                                                            WHERE ContactId =: userContactId AND (Roles INCLUDES ('PA Read', 'PA Write')
                                                                                                  OR IsDirect = True)]) 
            {
                setPAAccountIds.add(objAccConRelation.AccountId);    
            }
            
            whereConditionQueryString += ' ( CommunitySource__c = :strPA AND AccountId IN :setPAAccountIds ) OR';
            enteredConditions = true;
            
        }
        
        if((setCommunitySource.contains(SHOPSE) || setCommunitySource.isEmpty()) && extendedAppsList.contains(SHOPSE))  
        {
            whereConditionQueryString += ' ( (CommunitySource__c = :SHOPSE  OR (CommunitySource__c = NULL AND CreatedDate >=: cutOffDateTime)) AND ContactId = :userContactId ) OR ';
        }
        if((setCommunitySource.contains(premiumSupport) || setCommunitySource.isEmpty())  && extendedAppsList.contains(premiumSupport)) {
            
            LIST<CTR_ValueChainPlayers__c> lstCVCPs = new LIST<CTR_ValueChainPlayers__c>();
            Map<Id,String> mapOfSerContractCaseAccess = new Map<Id,String>();
            LIST<Id> lstServIdsWithMyCases = new LIST<Id>();
            LIST<Id> lstServIdsWithAccCases = new LIST<Id>();
            LIST<Id> lstServIdsWithContractCases = new LIST<Id>();
            
            lstCVCPs = getPremiumSupportContracts(loggedInUser);
            if(lstCVCPs.size()>0){
                for(CTR_ValueChainPlayers__c record : lstCVCPs){
                    if(record.ServiceMaintenanceContract__c != null){
                        cvcpServiceContractIds.add(record.ServiceMaintenanceContract__c);
                    }
                    mapOfSerContractCaseAccess.put(record.ServiceMaintenanceContract__c,record.CaseAccess__c);
                }
                
                mapIsPremiumSupport = WS_OCMAPIHandler.isServiceContractPremium(cvcpServiceContractIds);
                
                System.debug('mapIsPremiumSupport' + mapIsPremiumSupport);
                for(String servContractId : mapIsPremiumSupport.keySet() ){
                    if(mapIsPremiumSupport.get(servContractId) && mapOfSerContractCaseAccess.containsKey(servContractId)){
                        if(mapOfSerContractCaseAccess.get(servContractId) == 'My Cases')
                            lstServIdsWithMyCases.add(servContractId);
                        if(mapOfSerContractCaseAccess.get(servContractId) == 'My Account Cases')
                            lstServIdsWithAccCases.add(servContractId);
                        if(mapOfSerContractCaseAccess.get(servContractId) == 'All Contract Cases')
                            lstServIdsWithContractCases.add(servContractId);
                    }
                }
                String psWhereCondition = '';
                String finalStr = '';
                
                whereConditionQueryString += ' (CommunitySource__c = : premiumSupport ';
                
                if(lstServIdsWithMyCases.size()>0 || (lstServIdsWithAccCases.size()>0) || lstServIdsWithContractCases.size()>0){
                    whereConditionQueryString += 'AND (';
                }
                
                if(lstServIdsWithMyCases.size()>0)
                    psWhereCondition = ' (ContactId = :userContactId AND ServiceLine__c IN :lstServIdsWithMyCases) ';
                
                if(lstServIdsWithAccCases.size()>0 )
                    psWhereCondition +=  'OR (AccountId = :userAccountId AND ServiceLine__c IN :lstServIdsWithAccCases) ';
                
                if(lstServIdsWithContractCases.size()>0)
                    psWhereCondition +=  'OR (ServiceLine__c IN :lstServIdsWithContractCases)';
                
                if(psWhereCondition.startsWith('OR'))
                    finalStr = psWhereCondition.removeStartIgnoreCase('OR');
                else 
                    finalStr = psWhereCondition;
                System.debug('psWhereCondition after removing OR' + finalStr);
                
                whereConditionQueryString += finalStr;
                if(lstServIdsWithMyCases.size()>0 || (lstServIdsWithAccCases.size()>0) || lstServIdsWithContractCases.size()>0){
                    whereConditionQueryString += ')';
                }
                whereConditionQueryString += ') OR';
                enteredConditions = true;
            }
        }
        else{
            if((setCommunitySource.contains(premiumSupport) || setCommunitySource.isEmpty())  && extendedAppsList.contains(premiumSupport) && userAccountId!=null && userContactId!=null){
                whereConditionQueryString += ' ( CommunitySource__c = : premiumSupport AND AccountId = :userAccountId ) OR';
                enteredConditions = true;
            }
        }
        System.debug('!!!! whereConditionQueryString : ' + whereConditionQueryString );
        
        // whereConditionQueryString Manipulations start
        whereConditionQueryString = whereConditionQueryString.removeEndIgnoreCase('OR') ;                                           
        System.debug('!!!! whereConditionQueryString OR Removed : ' + whereConditionQueryString );
        
        whereConditionQueryString += ' )'; 
        
        whereConditionQueryString = whereConditionQueryString.removeEndIgnoreCase('AND (  )');
        System.debug('!!!! whereConditionQueryString AND ( ) Removed : ' + whereConditionQueryString );
        // whereConditionQueryString Manipulations end
        
        // Pagination starts
        
        if(String.isNotBlank(sortedBy)) {
            if(String.isNotBlank(mapOCMCustomSetting.SortByValue__c)){
                sortedBy = mapOCMCustomSetting.SortByValue__c;
            }    
        }
        else{
            sortedBy = 'CaseNumber';
        }                                                   
        whereConditionQueryString += ' ORDER BY '+ sortedBy;
        countCasesString = countCasesString + whereConditionQueryString;
        
        if(pageSize != null) {
            if(order != null && order.containsIgnoreCase('Asc')){
                whereConditionQueryString += ' '+order +' NULLS FIRST ' +' LIMIT '+ pageSize;
            }    
            else{
                whereConditionQueryString += ' '+'Desc' +' NULLS LAST ' +' LIMIT '+ pageSize;
            }
        }
        
        
        if(pageNumber != null && pageNumber > 1) {
            Integer pageNo = pageNumber-1;
            whereConditionQueryString += ' OFFSET '+ pageSize*pageNo;
        }
        
        // Pagination ends        
        String finalCaseListQueryString =  caseListQueryString + whereConditionQueryString; 
        
        if(enteredConditions){
            lstApiCases = Database.query(finalCaseListQueryString);
        }
        System.debug('!!!! lstApiCases : ' + lstApiCases);
        
        System.debug('!!!! countCasesString : ' + countCasesString);
        
        if (String.isNotBlank(countCasesString) && enteredConditions) {
            
            lstAllCases = Database.query(countCasesString) ;
            System.debug('!!!! lstAllCases : ' + lstAllCases);
            
            //gives total number of cases
            totalCaseCount = lstAllCases.size();
            System.debug('!!!! totalCaseCountInside : ' + totalCaseCount);
            
            if(pageSize != null) {
                totalNumberPage = totalCaseCount/pageSize;
                totalNumberOfPages = Integer.valueOf(totalNumberPage.round(System.RoundingMode.CEILING));
                totalNumberOfCases = Integer.valueOf(totalCaseCount);
            }
        }
        caseViewResWrapper.lstAPICases = lstApiCases;
        caseViewResWrapper.totalNumberOfPages = totalNumberOfPages;
        caseViewResWrapper.totalNumberOfCases = totalNumberOfCases;
        return caseViewResWrapper;
    }
    
    public static LIST<CTR_ValueChainPlayers__c> getPremiumSupportContracts(WS_OKTAAPIHandler.OktaGetUserWrapper loggedInUser){
        
        LIST<CTR_ValueChainPlayers__c> lstCVCPs = new LIST<CTR_ValueChainPlayers__c>();
        LIST<String> contactRoles = new LIST<String>();
        contactRoles.add(Label.CLSEP12CCC23);
        contactRoles.add(Label.CLOCT12CCC22);
        contactRoles.add('Inactive');
        contactRoles.add('No Access');
        contactRoles.add(Label.CLR0422Reseller);
        lstCVCPs = [SELECT Id,ServiceMaintenanceContract__c,CaseAccess__c FROM CTR_ValueChainPlayers__c 
                    WHERE Contact__c = :loggedInUser.ContactId
                    AND ContactRole__c IN : contactRoles AND CaseAccess__c != 'No Access'
                    AND CaseAccess__c != NULL];
        
        return lstCVCPs;
    }
    
    public static Boolean validateCommunitySource(Set<String> inCommunitySource){
        //get community source picklist values
        Schema.DescribeFieldResult objFieldDescribe = OCMApp__c.CommunitySource__c.getDescribe();
        List<Schema.PicklistEntry> lstPickListValues = objFieldDescribe.getPickListValues();
        Set<String> listCommunitySource =  new Set<String>();
        
        for (Schema.PicklistEntry objPickList : lstPickListValues) {
            listCommunitySource.add(objPickList.getValue());
        }
        
        //validate input aginst the community source picklist values
        Boolean validCommunitySource = false;
        for(String communitysource : inCommunitySource){
            communitysource = communitysource == 'OCM' ? 'Case Tracker' : communitysource == 'Case Tracker' ? '' : communitysource;
            validCommunitySource = listCommunitySource.contains(communitysource);
        }
        return validCommunitySource;
    }
    
    public static String getStatusFilterQueryString(Set<String> statusFilters, Id userContactId, String caseTracker) {
    String statusFilterQueryString = '';
         
		 if (!statusFilters.isEmpty()) {
        //CCCOCM-1632
        Map<String, String> statusConditions = new Map<String, String>{
            '01' => '((Status =:NEWSTATUS OR Status =:INPROGSTAUS) AND ActionNeededFrom__c != :CUSTOMER)',
            '02' => '((Status =:ANSPROVIDEDTOCUSTOMER OR ActionNeededFrom__c = :CUSTOMER) AND (Status != :CLOSEDSTATUS)  AND (Status!=:CANCELLEDSTATUS))',
            '03' => ' Status =:CLOSEDSTATUS',
            '04' => '(Status =:NEWSTATUS OR Status =:INPROGSTAUS OR Status =:ANSPROVIDEDTOCUSTOMER OR Status =:CLOSEDSTATUS)',
            '11' => '((Status =:NEWSTATUS OR Status =:INPROGSTAUS) AND ActionNeededFrom__c != :CUSTOMER)',
            '12' => '((Status =:ANSPROVIDEDTOCUSTOMER OR ActionNeededFrom__c = :CUSTOMER) AND (Status != :CLOSEDSTATUS))',
            '13' => 'Status =:CLOSEDSTATUS',
            '14' => '(Status =:NEWSTATUS OR Status =:INPROGSTAUS OR Status =:ANSPROVIDEDTOCUSTOMER OR Status =:CLOSEDSTATUS)'
        };
		
        statusFilterQueryString += ' AND (';

        for (String filter : statusFilters) {
            if (statusConditions.containsKey(filter)) {
                if (!filter.startsWith('0')) { // Personal cases
                    statusFilterQueryString += ' ';
                }
                statusFilterQueryString +=  statusConditions.get(filter) + ' OR';
            }
        }

        if (statusFilterQueryString.endsWith('OR')) {
            statusFilterQueryString = statusFilterQueryString.substring(0, statusFilterQueryString.length() - 3) + ')';
        }
        
    }
    return statusFilterQueryString;
}
    
public static Map<Id,String> getCustomFacingCategory(Set<string> caseCategoryValueMap,Set<string> caseReasonValueMap,String language,List<Case> caseLst){
    Map<String,List<CategoryLabels__c>> caseCatgeoryLabelMap = new Map<String,List<CategoryLabels__c>>(); 
    Map<Id,String> mapCategoryTranslation = new Map<Id,String>();
    List<CategoryLabels__c> categoryLabelLst = [select Id,bFOCaseCategory__c,bFOCaseReason__c,CommunitySource__c,(select id, UserLanguageCode__c, TranslatedValue__c from Category_Translations__r where UserLanguageCode__c = :language) from CategoryLabels__c where (bFOCaseCategory__c in :caseCategoryValueMap OR bFOCaseReason__c in  :caseReasonValueMap) and ApplicableToCaseList__c = true];        
    for(CategoryLabels__c cat : categoryLabelLst){
        if(caseCatgeoryLabelMap.containsKey(cat.bFOCaseCategory__c)){
            caseCatgeoryLabelMap.get(cat.bFOCaseCategory__c).add(cat); 
        }else{
            caseCatgeoryLabelMap.put(cat.bFOCaseCategory__c,new List<CategoryLabels__c>{cat}); 
        }
    }
    
    for(Case cs : caseLst){
        if(caseCatgeoryLabelMap.containsKey(cs.SupportCategory__c)){
            for(CategoryLabels__c cat : caseCatgeoryLabelMap.get(cs.SupportCategory__c)){
                if(cs.CommunitySource__c == 'Premium Support' && cat.CommunitySource__c == 'Premium Support'){
                    if(String.isNotBlank(cs.Symptom__c) && cs.Symptom__c == cat.bFOCaseReason__c){
                        for(CategoryTranslation__c catTrans : cat.Category_Translations__r){
                            mapCategoryTranslation.put(cs.Id,catTrans.TranslatedValue__c);
                        }
                    }else if(String.isBlank(cs.Symptom__c) || String.isBlank(cat.bFOCaseReason__c)){
                        for(CategoryTranslation__c catTrans : cat.Category_Translations__r){
                            mapCategoryTranslation.put(cs.Id,catTrans.TranslatedValue__c);
                        }
                    } 
                }else{
                    if(String.isNotBlank(cs.Symptom__c) && cs.Symptom__c == cat.bFOCaseReason__c){
                        for(CategoryTranslation__c catTrans : cat.Category_Translations__r){
                            mapCategoryTranslation.put(cs.Id,catTrans.TranslatedValue__c);
                        }
                    }else if(String.isBlank(cs.Symptom__c) || String.isBlank(cat.bFOCaseReason__c)){
                        for(CategoryTranslation__c catTrans : cat.Category_Translations__r){
                            mapCategoryTranslation.put(cs.Id,catTrans.TranslatedValue__c);
                        }
                    } 
                }    
            } 
        }
    }
    return mapCategoryTranslation;
}
    
    
    public static String validateLanguageCode (String language){
        
        Schema.DescribeFieldResult fieldResult = CategoryTranslation__c.UserLanguageCode__c.getDescribe();
        List<Schema.PicklistEntry> lsPicklistEntry = fieldResult.getPicklistValues();
        List<String> lsCountryCode = new List<String>();
        
        for(Schema.PicklistEntry ple : lsPicklistEntry){
            lsCountryCode.add(ple.getValue());
        }
        if (!lsCountryCode.contains(language)){
            language='en_US';
        }
        return language;
    }
    
    public class CaseViewResults {
        
        public List<Case> lstAPICases;
        public Integer totalNumberOfPages;
        public Integer totalNumberOfCases;
        public CaseViewResults() {}
        
    }
    
    global class CaseDetailWrapper {
        
        global String description;
        global String consumptionOfPoints;
        global String contractId;
        global String product;
        global String contractName;
        global String customerStatus;
        global String accountName;
        global String origin;
        global String accountId;
        global String caseResolvingAgentId;
        global String contactName;
        global String caseResolvingAgentName;
        global String contactId;
        global String answerProvidedToCustomer;
        global String subject;
        global String createdDate;
        global String communitySource;
        global String lastModifiedOn;
        global String caseNumber;
        global String closedDate;
        global String caseId;
        global String paAccountName;
        global String accountSEAccountId;
        global String paContactCfid;
        global String paCaseType;
        global String paHotCase;
        global String paCustomerInternalCaseNumber;
        global String paTargetDate;
        global String alarmDateAndTime;
        global String siteName;
        global String siteGuid;
        global String priority;
        global String firstLogicalResponseTargetDate;
        global String serviceContractId;
        global String serviceContractName;
        //Fields Added By SESA761027
        global String caseTracker;
        global String customerFacingCategory;
        global String customerRequest;
        global String installedAtAccountName;
        global String installedProductName;
        global String salesOrderNumber;       
        global String serviceLine;  
        
    }
    
    // wrapper class for response
    global class CaseListAPIResponse {
        
        global List<CaseDetailWrapper> cases;
        global Integer totalNumberOfPages;
        global Integer totalNumberOfCases;
    }
    // @CCC_SONARSCAN_START@
}
