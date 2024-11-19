sequenceDiagram
    participant User
    participant Frontend
    participant ChallengeController
    participant ChallengeService
    participant OtpSmsService
    participant OtpGridCardService
    participant OtpMailService
    participant Database
%% Step 1: Get Available OTP Types
    User ->> Frontend: GET /challenges/otp/{recipientToken}
    Frontend ->> ChallengeController: getAllOtp()
    ChallengeController ->> ChallengeService: getAllOtpTypes()
    ChallengeService -->> ChallengeController: Return OTP types (OTP_SMS, OTP_GRIDCARD, OTP_MAIL)
    ChallengeController -->> Frontend: Display OTP Types List
%% Step 2: Begin Challenge
    User ->> Frontend: Select OTP type (e.g., OTP_GRIDCARD)
    Frontend ->> ChallengeController: POST /challenges/begin/{recipientToken} with otpType
    ChallengeController ->> ChallengeService: beginChallengeWithOtp(recipientToken, otpType)
%% Check if OTP type is implicit and mode is direct
    alt OTP type is implicit and mode is direct
        ChallengeService ->> Database: Insert challengeToken into challenge_token table
        Database -->> ChallengeService: Return challengeToken
        ChallengeService -->> ChallengeController: Return challengeToken (direct mode)
        ChallengeController -->> Frontend: Display success
        Frontend ->> ChallengeController: Automatically submit challenge
    else
    %% Invoke appropriate OTP Service
        ChallengeService ->> OtpSmsService: performOtpSpecificActions() (if OTP_SMS)
        ChallengeService ->> OtpGridCardService: performOtpSpecificActions() (if OTP_GRIDCARD)
        ChallengeService ->> OtpMailService: performOtpSpecificActions() (if OTP_MAIL)
    %% Generate challengeToken
        ChallengeService ->> Database: Insert challengeToken into challenge_token table
        Database -->> ChallengeService: Return challengeToken
        ChallengeService -->> ChallengeController: Return challengeToken and data
        ChallengeController -->> Frontend: Display OTP prompt
    end

%% Step 3: Submit Challenge
    User ->> Frontend: Enter OTP and challengeToken
    Frontend ->> ChallengeController: POST /challenges/submit/{recipientToken}
    ChallengeController ->> ChallengeService: submitChallenge(recipientToken, otpCode, challengeToken)
%% Validate Token
    ChallengeService ->> Database: Validate challengeToken
    Database -->> ChallengeService: Return validity
    alt Token is valid
        ChallengeService -->> ChallengeController: Return success
        ChallengeController -->> Frontend: Notify success
        Frontend -->> User: Show success message
    else Token is invalid
        ChallengeService -->> ChallengeController: Return error
        ChallengeController -->> Frontend: Notify error
        Frontend -->> User: Show error message
    end
