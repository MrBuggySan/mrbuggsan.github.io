---
published: false
---


## Unit Test Definition
_"Essentially, a unit test is a method that instantiates a small portion of our application and verifies its behavior independently from other parts. A typical unit test contains 3 phases: First, it initializes a small piece of an application it wants to test (also known as the system under test, or SUT), then it applies some stimulus to the system under test (usually by calling a method on it), and finally, it observes the resulting behavior." - Toptal article on unit testing_

_"A unit test is an automated piece of code that invokes a unit of work in the system and then checks a single assumption about the behavior of that unit of work" - artofunittesting.com_

_From software product value perspective, we can summarize it as follows: unit testing is a practice which allows developers to add features with confidence._



## Action Items
* Write unit tests correctly
* Keep unit tests clean
* Unit testing as first class citizens


## The How

### Write unit tests correctly

#### Assert more 

Aim for TRUE test coverage first rather than creating unit tests to complete line coverage checks

 	--BAD - asserting a non null object is not enough
    @Test
    public void testMethod() {
        RequestDetails requestDetails = new RequestDetails();
        
        assertNotNull(controller.submitSelection(requestDetails));
    }
 
    --GOOD - the properties of the returned object is also asserted
    @Test
    public void testMethod {
        ConsolidatedResponse expectedResponse = mock(ConsolidatedResponse.class);
 
        ResponseEntity actualResponseEntity = controller.submitSelection(mock(requestDetails.class));
 
        assertEquals(HttpStatus.OK, actualResponseEntity.getStatusCode());
        assertEquals(expectedResponse, actualResponseEntity.getBody());
    }


#### Mock away dependencies of the unit in test
What is a unit? And It is not necessarily the class on its own with all dependencies mocked away
a piece of code which makes sense to be together therefore it is more valuable to test them together
ie controller related classes are tested together as a unit using SpringWebMvc
ie a class which maps between domain to entity objects should include the mapper in its unit

    --GOOD  - Everything but the mapper is mocked. This gives the programmer ease of use as they don't have to mock the mapping behavior
    --      - To ensure we are always using the correct mapper as production code, we could make a DozerBeanMapper factory which encodes mapping details. Then we could also create unit tests against that new class.
    public class RequestTrackerServiceImplTest {
 
        @InjectMocks
        RequestTrackerServiceImpl requestTrackerService;
        @Mock
        RequestTrackerRepository mockRequestTrackerRepository;
        @Mock
        IMPLogger mockImpLogger;
        private DozerBeanMapper beanMapper = new DozerBeanMapper(Arrays.asList("beanmaps/dictionary_map.xml", "beanmaps/segmentation_map.xml"));
        @Mock
        AudienceSegmentsManagementUtils utils;

    ...

    }
  - Any internal behavioral change of a dependency shouldn't break the unit test
  - Do not mock behavior of static methods
	
* Testing void methods
    //Unit in test - the method checks the type of the failure message, then calls appropriate method to process the failure message
    public void processFailureMessages(final FailureMessage failureMessage)
            throws JsonProcessingException, RequestTrackerServiceException, IAFRequestServiceException {
        String businessTrackerId = failureMessage.getHeader().getBusinessTrackerId();
 
        if (failureMessage.getRequestType() == RequestType.IAF) {
            iafRequestTrackerService.updateIafOnFail(failureMessage);
        } else {
            requestTrackerService.createRequestsForAxonMessage(
                    objectMapper.writeValueAsString(failureMessage),
                    businessTrackerId,
                    null,
                    failureMessage.getRequestType(),
                    failureMessage.getMessageType());
        }
    }
 
    //GOOD - given processFailureMessages when failure message is of type FEX then expected method is called
    @Test
    public void givenProcessFailureMessageWhenFexMessageThenExpectCorrectServicesCalled() throws RequestTrackerServiceException
            , JsonProcessingException, IAFRequestServiceException {
        String failureMessageAsString = "failureMessageAsString";
        when(objectMapper.writeValueAsString(any())).thenReturn(failureMessageAsString);
        messageProcessor.processFailureMessages(mockedAxonMessageFactory.createFailureMessage(RequestType.FEX));
        verify(requestTrackerService, times(1))
                .createRequestsForAxonMessage(eq(failureMessageAsString), any(String.class),
                        any(), eq(RequestType.FEX), eq(MessageType.FAILURE));
    }
Code Block 4 Use verify() for testing void methods

    //Unit in test - the method checks input nullability, maps values to requestTrackerEntity depending on a constraint, saves the entity in repo, maps the saved entity back to domain object and returns the domain object.
    public RequestTracker createRequestTracker(RequestTracker requestTracker) throws RequestTrackerServiceException {
        try {
            if (null != requestTracker && null!= requestTracker.getState()) {
                RequestTrackerEntity requestTrackerEntity = dozerBeanMapper
                    .map(requestTracker, RequestTrackerEntity.class, REQUEST_TRACKER_BEAN_MAP_ID);
                if (null != requestTracker.getParentRequestGuid()) {
                    RequestTrackerEntity parentRequestTracker = requestTrackerRepository
                        .getRequestTrackerById(requestTracker.getParentRequestGuid());
                    requestTrackerEntity.setParentRequestTracker(parentRequestTracker);
                    requestTrackerEntity.setCampaignGuid(parentRequestTracker.getCampaignGuid());
                }
                requestTrackerEntity.setStateCode(requestTracker.getState().state());
 
 
                    return dozerBeanMapper
                        .map(requestTrackerRepository.save(requestTrackerEntity), RequestTracker.class,
                            REQUEST_TRACKER_BEAN_MAP_ID);
            }else
                throw new RequestTrackerServiceException(ErrorCodes.REQUEST_TRACKER_SAVE_EXCEPTION);
        }catch (RuntimeException ex){
            logger.error(SEGMENTATION, ex);
            throw new RequestTrackerServiceException(ErrorCodes.REQUEST_TRACKER_SAVE_EXCEPTION, ex);
 
        }
    }
 
    //Current Impl - could be improved. This test does not verify whether the method has mapped requestTracker correctly
    @Test
    public void givenCreateRequestTrackerWhenRequestTrackerPassedAsExpectedThenMakeSureNewRequestTrackerIsCreatedAndReturned()
        throws  RequestTrackerServiceException {
        // given
        RequestTracker requestTracker = getRequestTracker();
        RequestTrackerEntity requestTrackerEntity = getRequestTrackerEntity(requestTracker);
        when(mockRequestTrackerRepository.save(any(RequestTrackerEntity.class))).thenReturn(requestTrackerEntity);
        // when
        RequestTracker requestTrackerResponse = requestTrackerService.createRequestTracker(requestTracker);
        // then
        assertNotNull(requestTrackerResponse);
        verify(mockRequestTrackerRepository).save(any(RequestTrackerEntity.class));
    }
 
    //BETTER - this is a better test since we capture the input to save() and assert whether the method has mapped the requestTrackerEntity as expected
    @Test
    public void givenCreateRequestTrackerWhenRequestTrackerPassedAsExpectedThenMakeSureNewRequestTrackerIsCreatedAndReturneds()
            throws  RequestTrackerServiceException {
        // given
        RequestTracker requestTracker = getRequestTracker();
        RequestTrackerEntity expectedRequestTrackerEntity = getRequestTrackerEntity(requestTracker);
        when(mockRequestTrackerRepository.save(any(RequestTrackerEntity.class))).thenReturn(expectedRequestTrackerEntity);
 
        // when
        RequestTracker actualRequestTrackerResponse = requestTrackerService.createRequestTracker(requestTracker);
 
        // then
        ArgumentCaptor<RequestTrackerEntity> argumentCaptor = ArgumentCaptor.forClass(RequestTrackerEntity.class);
        verify(mockRequestTrackerRepository).save(argumentCaptor.capture());
        RequestTrackerEntity actualEntitySaved = argumentCaptor.getValue();
        assertEquals(requestTracker.getState().state(), actualEntitySaved.getStateCode());
        assertEquals(expectedRequestTrackerEntity.getCampaignGuid(), actualRequestTrackerResponse.getCampaignGuid());
//        assert more details if possible....
    }
 
    //MUCH BETTER   - the test is efficiently mocking away save(), as we know save() returns the entity which was saved
    //              - we are now testing against the return value instead of using verify
    @Test
    public void givenCreateRequestTrackerWhenRequestTrackerPassedAsExpectedThenMakeSureNewRequestTrackerIsCreatedAndReturnedss()
            throws  RequestTrackerServiceException {
        RequestTracker requestTracker = getRequestTracker();
        when(mockRequestTrackerRepository.save(any())).thenAnswer(invocationOnMock -> invocationOnMock.getArgument(0, RequestTrackerEntity.class));
 
        RequestTracker actualRequestTrackerResponse = requestTrackerService.createRequestTracker(requestTracker);
 
        assertEquals(requestTracker.getCampaignGuid(), actualRequestTrackerResponse.getCampaignGuid());
        assertEquals(requestTracker.getState().state(), actualRequestTrackerResponse.getState().state());
        ...
        ...
    }
Code Block 5 Using verify is not always the case

* Test the error code
    //BAD   - if in the future updateIafRequestOnSuccess() does not throw an exception then this class would still pass
    //      - inside the catch we are not asserting the details of IAFRequestServiceException
    @Test(expected = IAFRequestServiceException.class)
    public void givenUpdateIafRequestOnSuccess_WhenChildRequestTrackerNotFound_ThenExpectExceptionThrown() throws IAFRequestServiceException {
        try {
            when(iafRequestDataAccessor.getIafRequestByRespKey(businessTrackerId)).thenThrow(IAFRequestServiceException.class);
            iafRequestTrackerServiceImpl.updateIafRequestOnSuccess(mockedSuccessMessage);
        } catch (IAFRequestServiceException e) {
            verify(iafRequestDataAccessor, never()).updateRequestState(eq(parentGuid), eq(IAFRequestState.IAF_COMPLETED));
            verify(iafRequestDataAccessor, never()).createRequestTracker(any(IAFRequestTracker.class));
            throw e;
        }
    }
 
    //GOOD  - fail the unit test right away if it doesn't throw an exception
    //      - assert the code inside IAFRequestServiceException
    @Test
    public void givenUpdateIafRequestOnSuccess_WhenChildRequestTrackerNotFound_ThenExpectExceptionThrowns() throws IAFRequestServiceException {
        when(iafRequestDataAccessor.getIafRequestByRespKey(businessTrackerId)).thenThrow(IAFRequestServiceException.class);
       
        try {
            iafRequestTrackerServiceImpl.updateIafRequestOnSuccess(mockedSuccessMessage);
           
            fail("must throw exception");
        } catch (IAFRequestServiceException e) {
            verify(iafRequestDataAccessor, never()).updateRequestState(eq(parentGuid), eq(IAFRequestState.IAF_COMPLETED));
            verify(iafRequestDataAccessor, never()).createRequestTracker(any(IAFRequestTracker.class));
            assertEquals(ErrorCodes.INVALID_AXON_MESSAGE, e.getCode());
        }
    }
Code Block 6 When an imp service exception is thrown, then verify the error code as well

* It matters how we create test data
    //BAD - this implementation runs the risk of tests depending on the specific values to pass. Also developers could unknowingly depend on the values of these values.
    public static Campaign mockCampaign() {
        Campaign campaign = new Campaign();
        campaign.setCampaignId(CAMPAIGN_ID);
        campaign.setCampaignName(CAMPAIGN_NAME);
        campaign.setPromoStartDate("05/01/2020");
        campaign.setPromoEndDate("05/01/2021");
        campaign.setClient(mockClient());
        campaign.setVendorGuid(VENDOR_GUID);
        campaign.setStatus(ACTIVE);
        campaign.setCampaignType(mockCampaignType());
        campaign.setCampaignOffer(mockCampaignOffer());
        campaign.setCampaignBinDetails(mockCampaignBinList());
        campaign.setPriority(1);
        return campaign;
    }
 
    //GOOD - rename the method name appropriately to reflect that static values are used
    public static Campaign mockDefaultCampaign() {
        Campaign campaign = new Campaign();
        campaign.setCampaignId(CAMPAIGN_ID);
        campaign.setCampaignName(CAMPAIGN_NAME);
        campaign.setPromoStartDate("05/01/2020");
        campaign.setPromoEndDate("05/01/2021");
        campaign.setClient(mockClient());
        campaign.setVendorGuid(VENDOR_GUID);
        campaign.setStatus(ACTIVE);
        campaign.setCampaignType(mockCampaignType());
        campaign.setCampaignOffer(mockCampaignOffer());
        campaign.setCampaignBinDetails(mockCampaignBinList());
        campaign.setPriority(1);
        return campaign;
    }
 
    //GOOD - use random values when possible
    public static Campaign mockCampaign() {
        Campaign campaign = new Campaign();
        campaign.setCampaignId(randomId());
        campaign.setCampaignName(randomName());
        campaign.setPromoStartDate("05/01/2020");
        campaign.setPromoEndDate("05/01/2021");
        campaign.setClient(rmockClient());
        campaign.setVendorGuid(radomGuid());
        campaign.setStatus(ACTIVE);
        campaign.setCampaignType(mockCampaignType());
        campaign.setCampaignOffer(mockCampaignOffer());
        campaign.setCampaignBinDetails(mockCampaignBinList());
        campaign.setPriority(1);
        return campaign;
    }
Code Block 7 When possible use randomized data

3.2 Keep unit tests clean
* prioritize readability
    //GOOD - white space provides clarity between the test's setup, call and assertion steps
    @Test
    public void testMethodName() {
        when(classDependency.doesMethod()).thenReturn(mock(SomeClassA.class);
       
        Response actualResp = classInTest.doMethod(getData());
           
        verify(classDependency, never()).someMethod(any());
        assertEquals(expectedId, actualResp.getId());
    }
    //MAYBE - could be useful but it could also be white noise
    //      - general rule is to only leave comments as part of a javadoc or when absolutely necessary. Leaving comments in code could lead to maintenance problems later on.
    @Test
    public void testMethodName() {
        //given
        when(classDependency.doesMethod()).thenReturn(mock(SomeClassA.class);
        //when
        Response actualResp = classInTest.doMethod(getData());
        //then    
        verify(classDependency, never()).someMethod(any());
        assertEquals(expectedId, actualResp.getId());
    }
Code Block 8 White space, white space, white space!

    //GOOD  - Although the name is long, it gives the reader information on the context and expectation of the test
    @Test
    givenCreateRequestTrackerWhenRequestTrackerPassedWithParentRequestTrackerThenMakeSureNewRequestTrackerIsCreatedAndReturned() {}
    @Test
    givenConvertIAFStateCodeToStateWithValidStateCodeThenSuccess() {}
    @Test
    givenCreateRequestTrackerWhenRequestTrackerPassedAsExpectedThenMakeSureNewRequestTrackerIsCreatedAndReturned() {}
Code Block 9 Test method naming convention: given<MethodName>When<Condition>Then<Expected_Behavior>

    //This test method is current implementation
    //TOO LONG - 11 mocks, on top of that the response is not asserted for its details. This is a code smell... we should refactor the unit in test
    // 1) This method is doing Feature Extraction in 4 steps
    // 2) This methid is doing Rules Submission 7 steps
    // 3) The actual mocked throwing of MultiValidationException isn't reached until the last mock
    // 4) Inside the catch where we assert details, it shows that the method is also explicitly responsible for setting the state to REJECTED
    // 5) If the method was modified sometime in the future where it does not throw an exception, this test would still pass. It now becomes a false positive since nothing was caught
    @Test
    public void givenSegmentationDetails_WhenSubmitSegmentationRequest_AndNoFexRecordInDb_ANdInvalidLookback_ThenBehavesAsExpected()
        throws FeatureExtractionServiceException, MultiValidationException, RequestTrackerServiceException, AudienceServiceException, DIPServiceException, InvalidSegmentException {
        // given
        SegmentationRequestDetails request1 = getSegmentationDetails(); // request with non-empty iafDueDate
        request1.setIafDueDate(null);// request with empty iafDueDate
        request1.setLookback(-1);
        CampaignInfo campaignInfo = createCampaignInfo();
        RequestTracker parentReq = createRequestTracker();
 
        // when - Feature Extraction
        when(audienceSegmentsManagementUtils.getCampaigns(anyListOf(String.class))).thenReturn(Collections.singletonList(campaignInfo));
        when(mockFeatureExtractionService.getExistingFeatureExtraction(any(FeatureExtraction.class), anyList())).thenReturn(null); // no pre-existing records in system
        when(mockRequestTrackerService.createRequestTracker(any(RequestTracker.class))).thenReturn(parentReq);
        when(mockDIPProxyService.triggerFeatureGeneration(any(DIPEventRequest.class))).thenReturn(new DIPBusinessTrackerResponse("BTID", "200"));
 
        // when - Rules Submission
        when(mockDIPProxyService.sendConfigurationRequest(anyString(), anyString(), anyString())).thenReturn(configResponsePayload);
        when(configResponsePayload.getDipConfigResponse()).thenReturn(dipConfigResponse);
        when(dipConfigResponse.getConfigFileId()).thenReturn("configFileId");
        when(configResponsePayload.getSegmentationRuleConfigurationRequest()).thenReturn(segmentationRuleConfigurationRequest);
        when(segmentationRuleConfigurationRequest.getPayload()).thenReturn("json");
        when(mockRequestTrackerService.updateRequestState(anyString(),anyInt())).thenReturn(true);
        when( audienceSegmentationValidator.validateRequest(Mockito.any())).thenThrow(MultiValidationException.class);
 
        // then
        try{
        SegmentationConsolidatedResponse response = audienceService.submitSegmentationRequest(request1);
 
            }catch (MultiValidationException ex) {
            verify(mockRequestTrackerService).createRequestTracker(requestTrackerArgumentCaptor.capture());
            assertEquals(AudienceRequestState.REJECTED.state(), requestTrackerArgumentCaptor.getValue().getState().state());
            verify(mockRequestTrackerService, times(1)).createRequestTracker(Mockito.any());
        }
    }
 
    //TODO refactor class AudienceServiceImpl or the method AudienceServiceImpl#SubmitSegmentationRequest()
    // Let's make AudienceService more of an orchestrator rather than having it handle too many details. This would mean pulling away a direct dependency to DIP proxy...
    // 1) Create a class to handle Feature Extraction logic with DIP
    // 2) Create a class to handle Config Rules Submission logic with DIP
    // 3) Create a class to handle Segmentation rule logic with DIP
    // 4) Create a class which handles failed requests coming from
    // ** making these type of refactoring changes is risky. As a general guidance it'd good to cover some of the main positive and negative paths using Spring integration testing.
Code Block 10 Keep it short

3.3 Unit testing as first class citizens
* Treat unit tests as production code
o Exert mostly the same effort in creating unit tests as creating implementation code
o On code reviews, we must review unit tests just like implementation code
*  Keep in mind FIRST* principles when creating tests
o  this image is from link
o  Notes:
* Timely***: it's imperative that we are writing testable code and we can only verify that when we write our unit tests the same time as we are implementing.
3.4 References:
* Clean Code by Robert C. Martin
* Clean Architecture by Robert C. Martin
* https://www.baeldung.com/mockito-behavior

"Test code is just as important as production code" - Robert C. Martin
"If you have tests you do not fear making changes to the code!" - Robert C. Martin
