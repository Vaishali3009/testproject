@PostMapping(value = "/cdp-api/prepMig",
            produces = {"application/json"},
            consumes = {"application/json"})
    ResponseEntity<ResultPrepMig> prepMig(@ApiParam(value = "Unique Request identifier", required = false) @RequestHeader(value = "RequestId", required = false) String requestId,
            @ApiParam(value = "Flag to insert data into CDP tables even if requested MSISDN is not available in Legacy dataset. This will decide whether to consider validation of MSISDN in Legacy dataset or not. 1 = Yes, 0 or EMPTY = No", required = false) @RequestHeader(value = "forcedUpdate", required = false) String forcedUpdate,
            @ApiParam(value = "Numerical MSISDN number. The MSISDN number have to start with prefix '49' e.g. 4917642042420", required = false) @Valid @RequestParam(value = "msisdn", required = false) String msisdn,
            @ApiParam(value = "Prepaid Customer object that needs to be added to the CDP database", required = true) @Valid @RequestBody PrepaidCustomer body) {
        String methodName = "prepMig";
        sessionID= UUID.randomUUID().toString();
        log.logEnterWebMethod(methodName, "sessionID ="+ sessionID +", requestId=" + requestId + ", msisdn=" + msisdn + ", body=" + body);
        long startTime = System.currentTimeMillis();
        
        long progressTime = 0;
        long blTime = 0;
        long dbTime = 0;

        // Validate request parameters and body
        ResponseEntity<ResultPrepMig> validationResponse = validatePrepMig(sessionID,methodName, startTime, ResponseUtils.neutralize(requestId), ResponseUtils.neutralize(msisdn), body);
        if (validationResponse != null) {
            return validationResponse;
        }

        // Set RequestId to response headers
        HttpHeaders responseHeaders = new HttpHeaders();
        responseHeaders.set("RequestId", ResponseUtils.neutralize(requestId));
        
        boolean updateCustomer = false;
        boolean dbResult = false;
        
        blTime = System.currentTimeMillis() - startTime;
        progressTime = System.currentTimeMillis();

        // Check if the customer exists in the Legacy DB (only for PPITPREPAID)
        if (body.getCustomerStackId() == PrepaidCustomer.CustomerStackIdEnum.PPITPREPAID && !"1".equals(forcedUpdate)) {
            ResponseEntity<ResultPrepMig> entity = isCustomerInLegacyDB(sessionID,methodName, startTime, ResponseUtils.neutralize(body.getMsisdn()), responseHeaders);
            if (entity != null) {
                // MSISDN not found in the Legacy DB, return error
                return entity;
            }
        }

        // Decide if the DB entry will be inserted or updated
        try {
            updateCustomer = Initializer.getCdpDb().isPrepaidCustomerInDB(sessionID,msisdn);
            if (log.isDebugEnabled()) {
                log.debug("sessionID = "+ sessionID+ ", Customer with MSISDN=" + msisdn + " exists in the DB (MC$_PREP_MIG_MSISDN).");
            }
       } catch (DBConnectionNotOpenException ex) {
            log.error("sessionID = "+ sessionID+ ", DB Connection is not opened:" + ex.getMessage());
            return createErrorResponse(sessionID,methodName, startTime, ResponseUtils.neutralize(msisdn), "Connection to the database failed.", 500, responseHeaders);
        } catch (SQLException ex) {
            log.error("sessionID = "+ sessionID+", Cannot get customer data from the Legacy DB. Exception:" + ex.getMessage());
            return createErrorResponse(sessionID,methodName, startTime, ResponseUtils.neutralize(msisdn), "Cannot get customer data from the DB.", 500, responseHeaders);
        }
        
        try {
            if (updateCustomer) {
                log.debug("sessionID = "+ sessionID+", Webmethod [" + methodName + "] - update customer with msisdn " + msisdn);
                dbResult = Initializer.getCdpDb().updateCustomerData(sessionID,msisdn, body);
            } else {
                log.debug("sessionID = "+ sessionID+", Webmethod [" + methodName + "] - store customer with msisdn " + msisdn);
                dbResult = Initializer.getCdpDb().storeCustomerData(msisdn, body,sessionID);
            }
        } catch (SQLException ex) {
            log.error("sessionID = "+ sessionID+", Exception catched in prepMig:" + ex.getMessage());
            return createErrorResponse(sessionID,methodName, startTime, ResponseUtils.neutralize(msisdn), "Cannot store or update customer data in the database.", 500, responseHeaders);
        }
        dbTime = System.currentTimeMillis() - progressTime;
        progressTime = System.currentTimeMillis();
        
        ResultPrepMig response = new ResultPrepMig();
        response.setMsisdn(msisdn);
        if (dbResult) {
            response.setResult("success");
        } else {
            response.setResult("failed");
        }
        
        blTime = blTime + (System.currentTimeMillis() - progressTime);
        log.debug("sessionID = "+ sessionID+" , MSISDN: " + msisdn + ", blTime=" + blTime + " ms, dbTime=" + dbTime + " ms");
        log.logExitWebMethod(methodName, 200, ", sessionID ="+ sessionID," Processing of customer with msisdn " + msisdn + " finished. Response:" + response, startTime);
        return new ResponseEntity<>(response, responseHeaders, HttpStatus.OK);
    }







public boolean updateCustomerData(String sessionID, String msisdn, PrepaidCustomer data) throws SQLException {
             if (log.isInfoEnabled()) {
                    log.debug("sessionID = "+ sessionID+", Entering updateCustomerData: msisdn=" + msisdn + ", data=" + data);
             }
             Boolean updatedSuccessfully = false;

             Connection conn = null;
             PreparedStatement stmt = null;

             String finalSql = "UPDATE MC$_PREP_MIG_MSISDN SET MSISDN = ?, CUSTOMER_STACK_ID = ?, CUSTOMER_ID = ?, CUSTOMER_TYPE = ?, BRANDID = ?,"
                           + " SPID = ?, PAYMENT_METHOD = ?, SALUT = ?, NAME = ?, FIRSTNAME = ?, LASTNAME = ?, TARIFF_NAME = ?, CONTRACT_ID = ?, ACTIVATE_DATE = ?,"
                           + " HAS_RECURRING_CHARGES = ?, EMAILADDR = ?, CACS_STATE = ?, TOTAL_DUE_AMOUNT = ?, UPD_DT = ? WHERE MSISDN = ?";

             final long startTime = new Date().getTime();

             try {
                    // Borrowing a connection from the pool
                    conn = cdpEtlPrepaidPds.getConnection();
                    // Working with the connection

                    if (conn != null) {
                           stmt = conn.prepareStatement(finalSql);

                           stmt.setString(1, data.getMsisdn()); // VARCHAR2(100)
                           if(data.getCustomerStackId() != null) {
                                  stmt.setString(2, data.getCustomerStackId().name()); // VARCHAR2(100)
                           } else {
                                  stmt.setNull(2, Types.VARCHAR);
                           }
                           stmt.setString(3, data.getCustomerId()); // VARCHAR2(100)
                           stmt.setString(4, data.getCustomerType()); // VARCHAR2(100)
                           stmt.setString(5, data.getBrandId()); // VARCHAR2(100)
                           stmt.setString(6, data.getSpId()); // VARCHAR2(100)
                           stmt.setString(7, data.getPaymentMethod()); // VARCHAR2(100)
                           stmt.setString(8, data.getSalut()); // VARCHAR2(100)
                           stmt.setString(9, data.getName()); // VARCHAR2(100)
                           stmt.setString(10, data.getFirstName()); // VARCHAR2(100)
                           stmt.setString(11, data.getLastName()); // VARCHAR2(100)
                           stmt.setString(12, data.getTariffname()); // VARCHAR2(100)
                           stmt.setString(13, data.getContractId()); // VARCHAR2(100)

                           SimpleDateFormat sdf = new SimpleDateFormat("dd.MM.yyyy");
                           java.sql.Date activationDate = null;
                           if (data.getActivationDate() != null) {
                                  try {
                                        Date tempDate = sdf.parse(data.getActivationDate());
                                        activationDate = new java.sql.Date(tempDate.getTime());
                                  } catch (ParseException e) {
                                        log.error("sessionID = "+ sessionID+", Activation date " + data.getActivationDate() + " is in worng format. Allowed format is 'dd.MM.yyyy'");
                                  }
                           }

                           if (activationDate == null) {
                                  stmt.setNull(14, Types.DATE);
                           } else {
                                  stmt.setDate(14, activationDate); // DATE
                           }

                           stmt.setString(15, data.getHasRecurringCharges()); // VARCHAR2(100)
                           stmt.setString(16, data.getEmailAddr()); // VARCHAR2(100)
                           stmt.setString(17, data.getCacsState()); // VARCHAR2(100)

                           Double dueAmount = null;
                           log.debug("sessionID = "+ sessionID+", checcking total amount");
                           if(data.getTotalDueAmount()!=null) {
                                  try {
                                        dueAmount = Double.parseDouble(data.getTotalDueAmount());
                                  } catch (Exception e) {
                                        log.error("sessionID = "+ sessionID+ ", TotalDueAmount " + data.getTotalDueAmount() + " cannot be parsed to a number.");
                                  }
                           }

                           if (dueAmount == null) {
                                  stmt.setNull(18, Types.DOUBLE);
                           } else {
                                  stmt.setDouble(18, dueAmount); // DATE
                           }

                           Timestamp timestamp = new Timestamp(System.currentTimeMillis());

                           stmt.setTimestamp(19, timestamp);
                           stmt.setString(20, data.getMsisdn());
                           stmt.executeUpdate();
                    }
                    updatedSuccessfully = true;
             } catch (final SQLException ex) {
                    log.error("sessionID = "+ sessionID+", Leaving updateCustomerData with SQLException: ", ex);
                    throw ex;
             } catch (Exception ex) {
                    log.error("sessionID = "+ sessionID+", Leaving updateCustomerData with unexpected exception: ", ex);
             } finally {
                    try {
                           if (stmt != null) {
                                  stmt.close();
                           }
                           if (conn != null) {
                                  conn.close();
                           }
                    } catch (final SQLException ex) {
                           log.error("sessionID = "+ sessionID+", SQLException while returning the db connection!", ex);
                    }
             }

             if (log.isInfoEnabled()) {
                    log.debug("sessionID = "+ sessionID+", Leaving updateCustomerData with result=" + updatedSuccessfully + ", duration: " + (System.currentTimeMillis() - startTime) + " ms");
             }
             return updatedSuccessfully;
       }


public boolean storeCustomerData(String msisdn, PrepaidCustomer data,String sessionID) throws SQLException {
             if (log.isInfoEnabled()) {
                    log.debug("sessionID = "+ sessionID+", Entering storeCustomerData: msisdn=" + msisdn + ", data=" + data);
             }
             final long startTime = System.currentTimeMillis();

             Boolean insertedSuccessfully = false;

             Connection conn = null;
             PreparedStatement stmt = null;
             String finalSql = "INSERT INTO MC$_PREP_MIG_MSISDN (MSISDN, CUSTOMER_STACK_ID, CUSTOMER_ID, CUSTOMER_TYPE, BRANDID,"
                           + " SPID, PAYMENT_METHOD, SALUT, NAME, FIRSTNAME, LASTNAME, TARIFF_NAME, CONTRACT_ID, ACTIVATE_DATE,"
                           + " HAS_RECURRING_CHARGES, EMAILADDR, CACS_STATE, TOTAL_DUE_AMOUNT, UPD_DT, INS_DT) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)";

             try {
                    // Borrowing a connection from the pool
                    conn = cdpEtlPrepaidPds.getConnection();
                    // Working with the connection

                    if (conn != null) {
                           stmt = conn.prepareStatement(finalSql);
                           stmt.setString(1, data.getMsisdn()); // VARCHAR2(100)
                           stmt.setString(2, data.getCustomerStackId().name()); // VARCHAR2(100)
                           stmt.setString(3, data.getCustomerId()); // VARCHAR2(100)
                           stmt.setString(4, data.getCustomerType()); // VARCHAR2(100)
                           stmt.setString(5, data.getBrandId()); // VARCHAR2(100)
                           stmt.setString(6, data.getSpId()); // VARCHAR2(100)
                           stmt.setString(7, data.getPaymentMethod()); // VARCHAR2(100)
                           stmt.setString(8, data.getSalut()); // VARCHAR2(100)
                           stmt.setString(9, data.getName()); // VARCHAR2(100)
                           stmt.setString(10, data.getFirstName()); // VARCHAR2(100)
                           stmt.setString(11, data.getLastName()); // VARCHAR2(100)
                           stmt.setString(12, data.getTariffname()); // VARCHAR2(100)
                           stmt.setString(13, data.getContractId()); // VARCHAR2(100)

                           SimpleDateFormat sdf = new SimpleDateFormat("dd.MM.yyyy");
                           java.sql.Date activationDate = null;
                           if (data.getActivationDate() != null) {
                                  try {
                                        Date tempDate = sdf.parse(data.getActivationDate());
                                        activationDate = new java.sql.Date(tempDate.getTime());
                                  } catch (ParseException e) {
                                        log.error("sessionID = "+ sessionID+", Activation date " + data.getActivationDate() + " is in worng format. Allowed format is 'dd.MM.yyyy'");
                                  }
                           }

                           if (activationDate == null) {
                                  stmt.setNull(14, Types.DATE);
                           } else {
                                  stmt.setDate(14, activationDate); // DATE
                           }

                           stmt.setString(15, data.getHasRecurringCharges()); // VARCHAR2(100)
                           stmt.setString(16, data.getEmailAddr()); // VARCHAR2(100)
                           stmt.setString(17, data.getCacsState()); // VARCHAR2(100)

                           Double dueAmount = null;
                           try {
                                  dueAmount = Double.parseDouble(data.getTotalDueAmount());
                           } catch (Exception e) {
                                  log.error("sessionID = "+ sessionID+", TotalDueAmount " + data.getTotalDueAmount() + " cannot be parsed to a number.");
                           }

                           if (dueAmount == null) {
                                  stmt.setNull(18, Types.DOUBLE);
                           } else {
                                  stmt.setDouble(18, dueAmount); // DATE
                           }

                           Timestamp timestamp = new Timestamp(System.currentTimeMillis());

                           stmt.setTimestamp(19, timestamp);
                           stmt.setTimestamp(20, timestamp);
                           stmt.executeUpdate();
                           insertedSuccessfully = true;
                    }
             } catch (final SQLException ex) {
                    log.error("sessionID = "+ sessionID+", Leaving storeCustomerData with SQLException: ", ex);
                    throw ex;
             } catch (Exception ex) {
                    log.error("sessionID = "+ sessionID+", Leaving storeCustomerData with unexpected exception: ", ex);
             } finally {
                    try {
                           if (stmt != null) {
                                  stmt.close();
                           }
                           if (conn != null) {
                                  conn.close();
                           }
                    } catch (final SQLException ex) {
                           log.error("sessionID = "+ sessionID+", SQLException while returning the db connection!", ex);
                    }
             }

             if (log.isInfoEnabled()) {
                    log.info("sessionID = "+ sessionID+", Leaving storeCustomerData with result=" + insertedSuccessfully + ", duration: " + (System.currentTimeMillis() - startTime) + " ms");
             }
             return insertedSuccessfully;
       }


  private ResponseEntity<ResultPrepMig> isCustomerInLegacyDB(String sessionID,String methodName, long startTime, String msisdn, HttpHeaders responseHeaders) {
        ResponseEntity<ResultPrepMig> response = null;
        try {
            boolean customerExistsInDB = Initializer.getCdpDb().isPrepaidCustomerInLegacyDB(msisdn,sessionID);
            if (!customerExistsInDB) {
                log.warn("sessionID = "+ sessionID+" , Customer with MSISDN=" + msisdn + " not found in the Legacy DB (CDP$_PEPS_LOOKUP_MSISDN_STATIC).");
                response = createErrorResponse(sessionID,methodName, startTime, msisdn,
                        "Request rejected - MSISDN not existing in legacy data (PPIT).", 422, responseHeaders);
            } else {
                log.debug("sessionID = "+ sessionID+" ,Customer with MSISDN=" + msisdn + " exists in the Legacy DB (CDP$_PEPS_LOOKUP_MSISDN_STATIC).");
            }
            
        } catch (DBConnectionNotOpenException ex) {
            log.error("sessionID = "+ sessionID+", message= DB Connection is not opened:" + ex.getMessage());
            response = createErrorResponse(sessionID,methodName, startTime, msisdn, "Connection to the database failed.",
                    HttpStatus.INTERNAL_SERVER_ERROR.value(), responseHeaders);
        } catch (SQLException ex) {
            log.error("sessionID = "+ sessionID+", message= Cannot get customer data from the Legacy DB. Exception:" + ex.getMessage());
            response = createErrorResponse(sessionID,methodName, startTime, msisdn, "Cannot get customer data from the Legacy DB.",
                    HttpStatus.INTERNAL_SERVER_ERROR.value(), responseHeaders);
        }
        return response;
    }

public boolean isPrepaidCustomerInDB(String sessionID, String msisdn) throws DBConnectionNotOpenException, SQLException {
             if (log.isInfoEnabled()) {
                    log.debug("sessionID = "+ sessionID+", [isPrepaidCustomerInDB] IN: msisdn=" + msisdn);
             }

             if (cdpEtlPrepaidPds == null) {
                    log.error("sessionID = "+ sessionID+", message="+connectionNotOpened);
                    throw new DBConnectionNotOpenException(connectionNotOpened);
             }

             final long startTime = new Date().getTime();
             boolean result = isMsisdnInDbTable(msisdn, "MC$_PREP_MIG_MSISDN",sessionID);

             if (DbConnector.log.isInfoEnabled()) {
                    DbConnector.log.debug("sessionID = "+ sessionID+", [isPrepaidCustomerInDB] for msisdn=" + msisdn + " Output: result=" + result
                                  + " , Duration:" + (new Date().getTime() - startTime) + " ms.");
             }

             return result;
       }

private boolean isMsisdnInDbTable(String msisdn, String tableName,String sessionID) throws SQLException {
             Connection conn = null;
             PreparedStatement stmt = null;
             ResultSet rs = null;
             boolean result = false;
             try {
                    // Borrowing a connection from the pool
                    conn = cdpEtlPrepaidPds.getConnection();
                    // Working with the connection

                    if (conn != null) {
                           stmt = conn.prepareCall("SELECT MSISDN FROM  " + tableName + " WHERE MSISDN=?");
                           stmt.setQueryTimeout(queryTimeoutSeconds * 1000);
                           stmt.setString(1, msisdn);
                           rs = stmt.executeQuery();
                           result = rs.next();
                    }
             } catch (SQLException eSQL) {
                    log.error("sessionID = "+ sessionID+", SQLException catched in isMsisdnInDbTable!", eSQL);

                    if (rs != null) {
                           rs.close();
                    }
                    if (stmt != null) {
                           stmt.close();
                    }
                    if (conn != null) {
                           conn.close();
                    }
                    conn = null;
                    evaluateSQLException(eSQL,sessionID);
                    throw eSQL;
             } catch (Exception e) {
                    log.error("sessionID = "+ sessionID+", Exception catched in isMsisdnInDbTable!", e);
             } finally {
                    if (rs != null) {
                           rs.close();
                    }
                    if (stmt != null) {
                           stmt.close();
                    }
                    if (conn != null) {
                           conn.close();
                           conn = null;
                    }
             }
             return result;
       }
