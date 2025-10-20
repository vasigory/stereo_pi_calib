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

фафваф


          
          // --- DEBUGGED chessboard detection block ---
          cv::Mat imgL = cv::imread(leftName);
          cv::Mat imgR = cv::imread(rightName);
          if (imgR.empty() || imgL.empty())
          {
              fprintf(stderr, "Pair No %d: cannot open images L('%s') R('%s')\n", photo_counter, leftName.c_str(), rightName.c_str());
              continue;
          }
          
          // Print sizes
          fprintf(stderr, "Pair No %d: Loaded images L=%dx%d R=%dx%d\n", 
                  photo_counter, imgL.cols, imgL.rows, imgR.cols, imgR.rows);
          
          // Convert to gray
          cv::Mat grayL_full, grayR_full;
          cv::cvtColor(imgL, grayL_full, cv::COLOR_BGR2GRAY);
          cv::cvtColor(imgR, grayR_full, cv::COLOR_BGR2GRAY);
          
          // Resize to working resolution (если нужно) — и используем уменьшенные версии для поиска углов
          cv::resize(grayL_full, gray_small_left, cv::Size(img_width, img_height), cv::INTER_AREA);
          cv::resize(grayR_full, gray_small_right, cv::Size(img_width, img_height), cv::INTER_AREA);
          
          // Try find on resized images first (more robust if sizes differ)
          std::vector<cv::Vec2f> cornersL_small, cornersR_small;
          int fbFlags = cv::CALIB_CB_ADAPTIVE_THRESH | cv::CALIB_CB_NORMALIZE_IMAGE; // без FAST_CHECK сначала
          bool retL_small = cv::findChessboardCorners(gray_small_left, CHECKERBOARD, cornersL_small, fbFlags);
          bool retR_small = cv::findChessboardCorners(gray_small_right, CHECKERBOARD, cornersR_small, fbFlags);
          
          // Debug prints
          fprintf(stderr, "  find on resized: retL=%d retR=%d cornersL=%zu cornersR=%zu\n",
                  (int)retL_small, (int)retR_small, cornersL_small.size(), cornersR_small.size());
          
          // If not found on resized, try on full resolution (fallback)
          std::vector<cv::Vec2f> cornersL_full, cornersR_full;
          bool retL_full = false, retR_full = false;
          if (!retL_small || !retR_small)
          {
              int fbFlagsFull = cv::CALIB_CB_ADAPTIVE_THRESH | cv::CALIB_CB_NORMALIZE_IMAGE | cv::CALIB_CB_FAST_CHECK;
              retL_full = cv::findChessboardCorners(grayL_full, CHECKERBOARD, cornersL_full, fbFlagsFull);
              retR_full = cv::findChessboardCorners(grayR_full, CHECKERBOARD, cornersR_full, fbFlagsFull);
              fprintf(stderr, "  find on full res fallback: retL=%d retR=%d cornersL=%zu cornersR=%zu\n",
                      (int)retL_full, (int)retR_full, cornersL_full.size(), cornersR_full.size());
          }
          
          // Choose which corners to use and scale if needed
          std::vector<cv::Vec2f> cornersL_used, cornersR_used;
          bool retL = false, retR = false;
          
          if (retL_small) {
              cornersL_used = cornersL_small;
              retL = true;
          } else if (retL_full) {
              // scale corners from full size to small working size
              double scale_x = (double)img_width / (double)grayL_full.cols;
              double scale_y = (double)img_height / (double)grayL_full.rows;
              cornersL_used = cornersL_full;
              for (auto &p : cornersL_used) { p[0] *= (float)scale_x; p[1] *= (float)scale_y; }
              retL = true;
          }
          
          if (retR_small) {
              cornersR_used = cornersR_small;
              retR = true;
          } else if (retR_full) {
              double scale_x = (double)img_width / (double)grayR_full.cols;
              double scale_y = (double)img_height / (double)grayR_full.rows;
              cornersR_used = cornersR_full;
              for (auto &p : cornersR_used) { p[0] *= (float)scale_x; p[1] *= (float)scale_y; }
              retR = true;
          }
          
          // Final debug counts
          fprintf(stderr, "  FINAL: retL=%d retR=%d usedCornersL=%zu usedCornersR=%zu\n", (int)retL, (int)retR, cornersL_used.size(), cornersR_used.size());
          
          // If both found and counts match expected -> refine and push
          if (retL && retR && cornersL_used.size() == (size_t)(CHECKERBOARD.width * CHECKERBOARD.height) && cornersR_used.size() == (size_t)(CHECKERBOARD.width * CHECKERBOARD.height))
          {
              cv::cornerSubPix(gray_small_left, cornersL_used, cv::Size(3,3), cv::Size(-1,-1), subpix_criteria);
              cv::cornerSubPix(gray_small_right, cornersR_used, cv::Size(3,3), cv::Size(-1,-1), subpix_criteria);
          
              objpointsLeft.push_back(objp);
              imgpointsLeft.push_back(cornersL_used);
              objpointsRight.push_back(objp);
              imgpointsRight.push_back(cornersR_used);
          
              fprintf(stderr, "  Pair No %d: ACCEPTED\n", photo_counter);
          }
          else
          {
              fprintf(stderr, "  Pair No %d: REJECTED (find failed or wrong count)\n", photo_counter);
          
              // Save debug image pair for manual inspection
              static const std::string dbgDir = folder_name + "debug_failed_pairs/";
              // create dir if not exists (POSIX)
              system((std::string("mkdir -p ") + dbgDir).c_str());
              char buf[256];
              snprintf(buf, sizeof(buf), "%s/left_%03d.png", dbgDir.c_str(), photo_counter);
              cv::imwrite(buf, imgL);
              snprintf(buf, sizeof(buf), "%s/right_%03d.png", dbgDir.c_str(), photo_counter);
              cv::imwrite(buf, imgR);
          
              continue;
          }




  Import pair No 32
  Pair No 32: Loaded images L=640x480 R=640x480
  find on resized: retL=1 retR=1 cornersL=54 cornersR=54
  FINAL: retL=1 retR=1 usedCornersL=54 usedCornersR=54
  Pair No 32: ACCEPTED

          
