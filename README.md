# stereo_pi_calib
terminate called after throwing an instance of 'cv::Exception'
  what():  OpenCV(4.1.1) /home/pi/opencv/modules/calib3d/src/fisheye.cpp:1023: error: (-215:Assertion failed) abs_max < threshold in function 'stereoCalibrate'



  
// Refine corners and add to array for processing
if (retL && retR)
{
    // Проверка согласованности размера
    if (cornersL.size() == cornersR.size() && cornersL.size() == objp.size())
    {
        cv::cornerSubPix(gray_small_left, cornersL, cv::Size(3,3), cv::Size(-1,-1), subpix_criteria);
        cv::cornerSubPix(gray_small_right, cornersR, cv::Size(3,3), cv::Size(-1,-1), subpix_criteria);

        objpointsLeft.push_back(objp);
        imgpointsLeft.push_back(cornersL);
        objpointsRight.push_back(objp);
        imgpointsRight.push_back(cornersR);
    }
    else
    {
        fprintf(stderr, "⚠️ Pair No %d skipped: inconsistent point count (L=%zu, R=%zu, Obj=%zu)\n",
                photo_counter, cornersL.size(), cornersR.size(), objp.size());
    }
}
else
{
    fprintf(stderr, "⚠️ Pair No %d ignored: chessboard not found on one or both cameras\n", photo_counter);
    continue;
}
